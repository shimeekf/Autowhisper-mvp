# Autowhisper-mvp
AutoWhisper – When your car talks, AutoWhisper listens.
import React, { useEffect, useRef, useState } from "react";
import { motion } from "framer-motion";
import { Tabs, TabsList, TabsTrigger, TabsContent } from "@/components/ui/tabs";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Badge } from "@/components/ui/badge";
import { Download, Mic, Square, Trash2, Wrench, MessageCircle, NotebookText, MapPin, Volume2, Settings, Plus, Upload } from "lucide-react";

// ------------------------------------------------------------
// AutoWhisper — Single-file React MVP
// Theme: black & orange; Assistant name: Axel
// Features:
// 1) Diagnose by sound (record/upload, simple heuristics)
// 2) Car logbook (service records, CSV export)
// 3) Chat with Axel (rule-based tips + text-to-speech)
// 4) Find nearby mechanics (mock list + geolocation distance)
// 5) Maintenance reminders (oil change) in a tiny Settings panel
// Storage: localStorage (prefix: aw__)
// ------------------------------------------------------------

const STORAGE = {
  logbook: "aw__logbook",
  chat: "aw__chat",
  profile: "aw__profile",
  favorites: "aw__favorites",
};

const defaultProfile = {
  owner: "",
  vehicle: "",
  lastOilChangeDate: "",
  lastOilChangeMileage: "",
  currentMileage: "",
  voiceVibe: "Confident",
};

function useLocalStorage(key, initial) {
  const [state, setState] = useState(() => {
    try {
      const v = localStorage.getItem(key);
      return v ? JSON.parse(v) : initial;
    } catch {
      return initial;
    }
  });
  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(state));
    } catch {}
  }, [key, state]);
  return [state, setState];
}

// -------------------------- Utilities --------------------------
function toCSV(rows) {
  if (!rows?.length) return "";
  const headers = Object.keys(rows[0]);
  const lines = [headers.join(",")].concat(
    rows.map((r) => headers.map((h) => `"${(r[h] ?? "").toString().replace(/"/g, '""')}"`).join(","))
  );
  return lines.join("\n");
}

function downloadFile(filename, content, mime = "text/plain") {
  const blob = new Blob([content], { type: mime });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = filename;
  a.click();
  URL.revokeObjectURL(url);
}

// Haversine distance in miles
function distanceMiles(a, b) {
  const R = 3958.8;
  const toRad = (d) => (d * Math.PI) / 180;
  const dLat = toRad(b.lat - a.lat);
  const dLon = toRad(b.lon - a.lon);
  const lat1 = toRad(a.lat);
  const lat2 = toRad(b.lat);
  const x = Math.sin(dLat / 2) ** 2 + Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLon / 2) ** 2;
  const y = 2 * Math.atan2(Math.sqrt(x), Math.sqrt(1 - x));
  return R * y;
}

// ------------------ Mock mechanic directory ------------------
const MOCK_MECHANICS = [
  { id: 1, name: "Axel's Trusted Auto Care", lat: 40.7357, lon: -74.1724, phone: "(973) 555-0141", rating: 4.8 },
  { id: 2, name: "Orange & Black Garage", lat: 40.742, lon: -74.18, phone: "(973) 555-0179", rating: 4.6 },
  { id: 3, name: "QuickFix Brakes & Lube", lat: 40.73, lon: -74.19, phone: "(973) 555-0190", rating: 4.2 },
];

// ----------------------- Diagnose Logic -----------------------
async function basicAnalyzeSound(blob) {
  // This is a lightweight heuristic analyzer meant for demo purposes.
  // It uses duration and loudness proxy to guess likely issues.
  const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  const arrayBuf = await blob.arrayBuffer();
  const audioBuf = await audioCtx.decodeAudioData(arrayBuf);
  const duration = audioBuf.duration; // seconds

  // Estimate average RMS over first channel
  const data = audioBuf.getChannelData(0);
  let sumSquares = 0;
  for (let i = 0; i < data.length; i += 1024) {
    const end = Math.min(i + 1024, data.length);
    let s = 0;
    for (let j = i; j < end; j++) s += data[j] * data[j];
    sumSquares += s / (end - i);
  }
  const rms = Math.sqrt(sumSquares / Math.ceil(data.length / 1024));

  // Heuristics
  // Short & loud pops -> possible exhaust backfire; long & whiny (medium rms, long duration) -> belt squeal
  // Very low rms, short duration -> normal click/idle; Medium duration + medium rms + repeating -> wheel bearing/alternator hum (approx)
  let label = "Unknown";
  let confidence = 0.4;
  let tips = [];

  if (duration < 1.2 && rms > 0.12) {
    label = "Possible exhaust backfire / misfire";
    confidence = 0.7;
    tips = [
      "Check for exhaust leaks and unburnt fuel.",
      "Scan for engine codes (P0300-P0308).",
      "Inspect spark plugs and ignition coils.",
    ];
  } else if (duration >= 1.2 && duration < 8 && rms >= 0.04 && rms < 0.12) {
    label = "Likely belt squeal (serpentine)";
    confidence = 0.72;
    tips = [
      "Check belt tension and glazing.",
      "Inspect pulleys and idler/tensioner bearings.",
      "Avoid spraying belt dressings; fix root cause.",
    ];
  } else if (duration >= 2 && rms < 0.04) {
    label = "Light ticking—could be lifter tick or normal injector noise";
    confidence = 0.55;
    tips = [
      "Verify oil level and viscosity.",
      "If persistent when warm, consider lifter inspection.",
      "Record again closer to source for better analysis.",
    ];
  } else if (duration >= 4 && rms >= 0.08) {
    label = "Deep hum—possible wheel bearing or alternator whine";
    confidence = 0.6;
    tips = [
      "Does pitch change with speed (bearing) or electrical load (alternator)?",
      "Check for play/noise when spinning wheels off-ground.",
      "Load-test alternator/battery.",
    ];
  } else {
    label = "No clear match—try again near the noise source";
    confidence = 0.35;
    tips = [
      "Record 5–10 seconds with phone 1–2 ft from source.",
      "Turn off radio/AC to reduce background noise.",
      "Note if noise changes with speed, RPM, or braking.",
    ];
  }

  await audioCtx.close();
  return { duration, rms: Number(rms.toFixed(3)), label, confidence, tips };
}

// ------------------------- Components -------------------------
function Header({ onOpenSettings }) {
  return (
    <div className="sticky top-0 z-20 border-b bg-black text-white">
      <div className="mx-auto max-w-6xl px-4 py-3 flex items-center justify-between">
        <div className="flex items-center gap-3">
          <div className="h-9 w-9 rounded-xl bg-orange-500 grid place-items-center font-bold">AW</div>
          <div>
            <div className="text-lg font-semibold">AutoWhisper</div>
            <div className="text-xs text-neutral-300">When your car talks, Axel listens.</div>
          </div>
        </div>
        <Button variant="ghost" className="text-orange-400 hover:text-orange-300" onClick={onOpenSettings}>
          <Settings className="mr-2 h-4 w-4" /> Settings
        </Button>
      </div>
    </div>
  );
}

function SettingsPanel({ open, onClose, profile, setProfile }) {
  if (!open) return null;
  const upcoming = computeOilReminder(profile);
  return (
    <div className="fixed inset-0 z-30 bg-black/60 backdrop-blur-sm grid place-items-center p-4">
      <Card className="w-full max-w-2xl">
        <CardHeader className="flex flex-row items-center justify-between">
          <CardTitle>Vehicle Profile & Reminders</CardTitle>
          <Button variant="ghost" onClick={onClose}>Close</Button>
        </CardHeader>
        <CardContent className="space-y-4">
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label className="text-sm">Owner</label>
              <Input value={profile.owner} onChange={(e) => setProfile({ ...profile, owner: e.target.value })} />
            </div>
            <div>
              <label className="text-sm">Vehicle (e.g., 2016 Honda Accord LX)</label>
              <Input value={profile.vehicle} onChange={(e) => setProfile({ ...profile, vehicle: e.target.value })} />
            </div>
            <div>
              <label className="text-sm">Last Oil Change Date</label>
              <Input type="date" value={profile.lastOilChangeDate} onChange={(e) => setProfile({ ...profile, lastOilChangeDate: e.target.value })} />
            </div>
            <div>
              <label className="text-sm">Last Oil Change Mileage</label>
              <Input type="number" value={profile.lastOilChangeMileage} onChange={(e) => setProfile({ ...profile, lastOilChangeMileage: e.target.value })} />
            </div>
            <div>
              <label className="text-sm">Current Mileage</label>
              <Input type="number" value={profile.currentMileage} onChange={(e) => setProfile({ ...profile, currentMileage: e.target.value })} />
            </div>
            <div>
              <label className="text-sm">Axel Voice Vibe</label>
              <select className="w-full border rounded-md h-10 px-3" value={profile.voiceVibe} onChange={(e) => setProfile({ ...profile, voiceVibe: e.target.value })}>
                <option>Confident</option>
                <option>Chill</option>
                <option>Coach</option>
                <option>Techy</option>
              </select>
            </div>
          </div>

          <div className="rounded-xl bg-neutral-50 p-4 border">
            <div className="font-medium mb-2">Oil Change Reminder</div>
            {upcoming?.due ? (
              <div className="text-red-600">Due now — {upcoming.reason}</div>
            ) : (
              <div className="text-green-700">Not due yet. {upcoming?.reason}</div>
            )}
          </div>
        </CardContent>
      </Card>
    </div>
  );
}
function computeOilReminder(profile) {

if (!profile) return null;
const daysSince = profile.lastOilChangeDate ? Math.floor((Date.now() - new Date(profile.lastOilChange).getTime()) / (1000 * 60 * 60 * 24)) : null;
const milesSince = profile.currentMileage && profile.lastOilChangeMileage ? Number(profile.currentMileage) - Number(profile.lastOilChangeMileage) : null;
const dueByDays = daysSince != null && daysSince >= 90;
const dueByMiles = milesSince != null && milesSince >= 5000;
const reason = [
daysSince != null ? ${daysSince} days since last oil change : null,
milesSince != null ? ${milesSince} miles since last oil change : null,
].filter(Boolean).join(" · ") || "Set your last oil change to enable reminders";
return { due: Boolean(dueByDays || dueByMiles), reason }; -} +function computeOilReminder(profile) {
if (!profile) return null;
// Safe guards for invalid/missing data
let daysSince = null;
try {
if (profile.lastOilChangeDate) {
 const t = new Date(profile.lastOilChangeDate).getTime();
 if (Number.isFinite(t)) {
   const delta = Date.now() - t;
   daysSince = Math.floor(delta / (1000 * 60 * 60 * 24));
 }
}
} catch {
daysSince = null;
}
let milesSince = null;
try {
const curr = Number(profile.currentMileage);
const last = Number(profile.lastOilChangeMileage);
if (Number.isFinite(curr) && Number.isFinite(last)) milesSince = curr - last;
} catch {
milesSince = null;
}
const dueByDays = Number.isFinite(daysSince) && daysSince >= 90;
const dueByMiles = Number.isFinite(milesSince) && milesSince >= 5000;
const parts = [];
if (Number.isFinite(daysSince)) parts.push(${daysSince} days since last oil change);
if (Number.isFinite(milesSince)) parts.push(${milesSince} miles since last oil change);
const reason = parts.length > 0 ? parts.join(" · ") : "Set your last oil change to enable reminders";
return { due: dueByDays || dueByMiles, reason }; +}


function DiagnoseTab() {
  const [recording, setRecording] = useState(false);
  const [mediaRecorder, setMediaRecorder] = useState(null);
  const chunksRef = useRef([]);
  const [audioURL, setAudioURL] = useState("");
  const [result, setResult] = useState(null);
  const [busy, setBusy] = useState(false);

  const startRecording = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      const mr = new MediaRecorder(stream);
      chunksRef.current = [];
      mr.ondataavailable = (e) => e.data.size && chunksRef.current.push(e.data);
      mr.onstop = async () => {
        const blob = new Blob(chunksRef.current, { type: "audio/webm" });
        const url = URL.createObjectURL(blob);
        setAudioURL(url);
        setBusy(true);
        try {
          const wavBlob = await ensureWav(blob);
          const analysis = await basicAnalyzeSound(wavBlob);
          setResult(analysis);
        } catch (e) {
          console.error(e);
          setResult({ label: "Analysis failed", confidence: 0, rms: 0, duration: 0, tips: ["Try recording again."] });
        } finally {
          setBusy(false);
        }
      };
      mr.start();
      setMediaRecorder(mr);
      setRecording(true);
    } catch (e) {
      alert("Microphone permission denied or unavailable.");
      console.error(e);
    }
  };

  const stopRecording = () => {
    if (mediaRecorder && mediaRecorder.state !== "inactive") {
      mediaRecorder.stop();
      mediaRecorder.stream.getTracks().forEach((t) => t.stop());
    }
    setRecording(false);
  };

  const onUpload = async (file) => {
    if (!file) return;
    setAudioURL(URL.createObjectURL(file));
    setBusy(true);
    try {
      const wavBlob = await ensureWav(file);
      const analysis = await basicAnalyzeSound(wavBlob);
      setResult(analysis);
    } catch (e) {
      console.error(e);
      setResult({ label: "Analysis failed", confidence: 0, rms: 0, duration: 0, tips: ["Try another clip."] });
    } finally {
      setBusy(false);
    }
  };

  return (
    <div className="grid gap-4 md:grid-cols-5">
      <Card className="md:col-span-3">
        <CardHeader>
          <CardTitle className="flex items-center gap-2"><Wrench className="h-5 w-5 text-orange-500" /> Diagnose by Sound</CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          <div className="flex items-center gap-3">
            {!recording ? (
              <Button onClick={startRecording} className="bg-orange-600 hover:bg-orange-500"><Mic className="mr-2 h-4 w-4" /> Record</Button>
            ) : (
              <Button onClick={stopRecording} variant="destructive"><Square className="mr-2 h-4 w-4" /> Stop</Button>
            )}
            <label className="inline-flex items-center gap-2 text-sm cursor-pointer">
              <Upload className="h-4 w-4" />
              <input type="file" accept="audio/*" className="hidden" onChange={(e) => onUpload(e.target.files?.[0])} />
              Upload clip
            </label>
          </div>

          {audioURL && (
            <div className="space-y-2">
              <audio controls src={audioURL} className="w-full" />
              <div className="text-xs text-neutral-500">If the clip is quiet, record closer to the source for better results.</div>
            </div>
          )}

          {busy && <div className="text-sm">Analyzing… give me a sec.</div>}

          {result && (
            <motion.div initial={{ opacity: 0, y: 8 }} animate={{ opacity: 1, y: 0 }} className="rounded-xl border p-4">
              <div className="flex items-center justify-between">
                <div className="font-semibold">{result.label}</div>
                <Badge variant="secondary">Confidence {Math.round(result.confidence * 100)}%</Badge>
              </div>
              <div className="mt-2 text-sm text-neutral-600">Duration: {result.duration.toFixed(2)}s · RMS: {result.rms}</div>
              <ul className="mt-3 list-disc pl-5 space-y-1 text-sm">
                {result.tips.map((t, i) => <li key={i}>{t}</li>)}
              </ul>
            </motion.div>
          )}
        </CardContent>
      </Card>

      <Card className="md:col-span-2">
        <CardHeader>
          <CardTitle className="flex items-center gap-2"><NotebookText className="h-5 w-5 text-orange-500" /> Quick Notes</CardTitle>
        </CardHeader>
        <CardContent>
          <QuickNotes />
        </CardContent>
      </Card>
    </div>
  );
}

function QuickNotes() {
  const [text, setText] = useState("");
  return (
    <div className="space-y-3">
      <Textarea rows={8} placeholder="Describe the noise, when it happens, temps, speed, braking, turning…" value={text} onChange={(e) => setText(e.target.value)} />
      <div className="text-xs text-neutral-500">Tip: Detailed notes help a mechanic diagnose faster and cheaper.</div>
    </div>
  );
}

async function ensureWav(blob) {
  // Convert any audio blob to PCM WAV via Web Audio (best-effort)
  try {
    const ctx = new (window.AudioContext || window.webkitAudioContext)();
    const ab = await blob.arrayBuffer();
    const buf = await ctx.decodeAudioData(ab);
    const wav = audioBufferToWav(buf);
    await ctx.close();
    return new Blob([wav], { type: "audio/wav" });
  } catch {
    return blob; // fallback
  }
}

function audioBufferToWav(buffer) {
  const numOfChan = buffer.numberOfChannels;
  const sampleRate = buffer.sampleRate;
  const format = 1; // PCM
  const bitDepth = 16;

  let result;
  if (numOfChan === 2) {
    const interleaved = interleave(buffer.getChannelData(0), buffer.getChannelData(1));
    result = encodeWAV(interleaved, numOfChan, sampleRate, bitDepth);
  } else {
    result = encodeWAV(buffer.getChannelData(0), 1, sampleRate, bitDepth);
  }
  return result;
}

function interleave(inputL, inputR) {
  const length = inputL.length + inputR.length;
  const result = new Float32Array(length);
  let index = 0, inputIndex = 0;
  while (index < length) {
    result[index++] = inputL[inputIndex];
    result[index++] = inputR[inputIndex];
    inputIndex++;
  }
  return result;
}

function encodeWAV(samples, numChannels, sampleRate, bitDepth) {
  const bytesPerSample = bitDepth / 8;
  const blockAlign = numChannels * bytesPerSample;
  const buffer = new ArrayBuffer(44 + samples.length * bytesPerSample);
  const view = new DataView(buffer);

  /* RIFF identifier */ writeString(view, 0, "RIFF");
  /* RIFF chunk length */ view.setUint32(4, 36 + samples.length * bytesPerSample, true);
  /* RIFF type */ writeString(view, 8, "WAVE");
  /* format chunk identifier */ writeString(view, 12, "fmt ");
  /* format chunk length */ view.setUint32(16, 16, true);
  /* sample format (raw) */ view.setUint16(20, 1, true);
  /* channel count */ view.setUint16(22, numChannels, true);
  /* sample rate */ view.setUint32(24, sampleRate, true);
  /* byte rate (sample rate * block align) */ view.setUint32(28, sampleRate * blockAlign, true);
  /* block align (channel count * bytes per sample) */ view.setUint16(32, blockAlign, true);
  /* bits per sample */ view.setUint16(34, bitDepth, true);
  /* data chunk identifier */ writeString(view, 36, "data");
  /* data chunk length */ view.setUint32(40, samples.length * bytesPerSample, true);

  if (bitDepth === 16) floatTo16BitPCM(view, 44, samples);
  else throw new Error("Only 16-bit PCM supported");

  return view;
}

function writeString(view, offset, string) {
  for (let i = 0; i < string.length; i++) {
    view.setUint8(offset + i, string.charCodeAt(i));
  }
}

function floatTo16BitPCM(output, offset, input) {
  for (let i = 0; i < input.length; i++, offset += 2) {
    let s = Math.max(-1, Math.min(1, input[i]));
    output.setInt16(offset, s < 0 ? s * 0x8000 : s * 0x7fff, true);
  }
}

function LogbookTab() {
  const [entries, setEntries] = useLocalStorage(STORAGE.logbook, []);
  const [form, setForm] = useState({ date: "", service: "", mileage: "", notes: "" });

  const addEntry = () => {
    if (!form.date || !form.service) return alert("Date and Service are required.");
    const newEntries = [{ id: crypto.randomUUID(), ...form }, ...entries];
    setEntries(newEntries);
    setForm({ date: "", service: "", mileage: "", notes: "" });
  };

  const removeEntry = (id) => setEntries(entries.filter((e) => e.id !== id));

 ;const exportCSV = () => {
const csv = toCSV(entries.map(({ id, ...rest }) => rest));
downloadFile("aw_logbook.csv", csv, "text/csv"); -};
};
const exportCSV = () => {
const csv = toCSV(entries.map(({ id, ...rest }) => rest));
downloadFile("aw_logbook.csv", csv, "text/csv");
}; 

  };

  return (
    <div className="grid gap-4 md:grid-cols-3">
      <Card className="md:col-span-1">
        <CardHeader>
          <CardTitle>Add Service Record</CardTitle>
        </CardHeader>
        <CardContent className="space-y-3">
          <div>
            <label className="text-sm">Date</label>
            <Input type="date" value={form.date} onChange={(e) => setForm({ ...form, date: e.target.value })} />
          </div>
          <div>
            <label className="text-sm">Service</label>
            <Input placeholder="Oil change, Brake pads, Tire rotation…" value={form.service} onChange={(e) => setForm({ ...form, service: e.target.value })} />
          </div>
          <div>
            <label className="text-sm">Mileage</label>
            <Input type="number" value={form.mileage} onChange={(e) => setForm({ ...form, mileage: e.target.value })} />
          </div>
          <div>
            <label className="text-sm">Notes</label>
            <Textarea rows={4} value={form.notes} onChange={(e) => setForm({ ...form, notes: e.target.value })} />
          </div>
          <Button className="w-full bg-orange-600 hover:bg-orange-500" onClick={addEntry}><Plus className="mr-2 h-4 w-4" /> Add</Button>
          <Button variant="secondary" className="w-full" onClick={exportCSV}><Download className="mr-2 h-4 w-4" /> Export CSV</Button>
        </CardContent>
      </Card>

      <Card className="md:col-span-2">
        <CardHeader>
          <CardTitle>History</CardTitle>
        </CardHeader>
        <CardContent className="space-y-3">
          {entries.length === 0 && <div className="text-sm text-neutral-600">No records yet.</div>}
          <div className="space-y-3">
            {entries.map((e) => (
              <div key={e.id} className="rounded-xl border p-3 flex items-start justify-between">
                <div>
                  <div className="font-medium">{e.service}</div>
                  <div className="text-xs text-neutral-600">{e.date} · {e.mileage ? `${e.mileage} mi` : "no mileage"}</div>
                  {e.notes && <div className="mt-1 text-sm">{e.notes}</div>}
                </div>
                <Button size="icon" variant="ghost" onClick={() => removeEntry(e.id)}><Trash2 className="h-4 w-4" /></Button>
              </div>
            ))}
          </div>
        </CardContent>
      </Card>
    </div>
  );
}

function ChatTab({ profile }) {
  const [messages, setMessages] = useLocalStorage(STORAGE.chat, [
    { role: "assistant", text: `Yo, I'm Axel — your ${profile.voiceVibe?.toLowerCase() || "confident"} virtual mechanic. What noise are we hunting today?` },
  ]);
  const [input, setInput] = useState("");
  const [speaking, setSpeaking] = useState(false);

  const speak = (text) => {
    if (!("speechSynthesis" in window)) return;
    const u = new SpeechSynthesisUtterance(text);
    u.rate = profile.voiceVibe === "Chill" ? 0.95 : profile.voiceVibe === "Coach" ? 1.1 : 1;
    u.pitch = profile.voiceVibe === "Techy" ? 1.2 : 1;
    setSpeaking(true);
    u.onend = () => setSpeaking(false);
    window.speechSynthesis.speak(u);
  };

  const reply = (q) => {
    const lower = q.toLowerCase();
    let a = "I couldn't catch that. Tell me when it happens: cold start, turning, braking, or highway speeds?";
    if (/(oil|change)/.test(lower)) a = "Oil game 101: every ~5,000 miles or 3 months. Check your dipstick on level ground; if it's dark/low, change it and the filter.";
    else if (/(battery|alternator)/.test(lower)) a = "If headlights dim at idle but brighten with revs, alternator might be weak. Get a load test (12.6V off, ~14V running).";
    else if (/(brake|squeal|grind)/.test(lower)) a = "Squeal usually means wear indicators; grind means pad gone — park it and replace pads/rotors ASAP.";
    else if (/(overheat|coolant|radiator)/.test(lower)) a = "Let it cool, check coolant reservoir. If it drops fast, pressure test for leaks; also check the fan and thermostat.";
    else if (/(belt|squeal)/.test(lower)) a = "Likely serpentine belt or tensioner. Inspect for glazing/cracks; replace belt and check pulley bearings.";
    else if (/(wheel|bearing|hum|whine)/.test(lower)) a = "If the hum grows with speed and changes in turns, front wheel bearing may be shot. Jack it up and spin — any roughness = replace.";

    return a;
  };

  const send = () => {
    if (!input.trim()) return;
    const userMsg = { role: "user", text: input.trim() };
    const botText = reply(input.trim());
    const botMsg = { role: "assistant", text: botText };
    const newMsgs = [...messages, userMsg, botMsg];
    setMessages(newMsgs);
    setInput("");
    speak(botText);
  };

  return (
    <div className="grid gap-4 md:grid-cols-3">
      <Card className="md:col-span-2">
        <CardHeader>
          <CardTitle className="flex items-center gap-2"><MessageCircle className="h-5 w-5 text-orange-500" /> Chat with Axel</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="h-[420px] overflow-y-auto space-y-3 pr-1">
            {messages.map((m, i) => (
              <div key={i} className={`max-w-[85%] rounded-2xl p-3 ${m.role === "assistant" ? "bg-neutral-100" : "bg-orange-600 text-white ml-auto"}`}>
                {m.text}
              </div>
            ))}
          </div>
          <div className="mt-3 flex gap-2">
            <Input placeholder="Describe the noise or ask a question…" value={input} onChange={(e) => setInput(e.target.value)} onKeyDown={(e) => e.key === "Enter" && send()} />
            <Button className="bg-orange-600 hover:bg-orange-500" onClick={send}>Send</Button>
            <Button variant="outline" onClick={() => window.speechSynthesis?.cancel()} disabled={!speaking}><Volume2 className="h-4 w-4" /></Button>
          </div>
        </CardContent>
      </Card>

      <Card>
        <CardHeader>
          <CardTitle>Tips</CardTitle>
        </CardHeader>
        <CardContent className="text-sm space-y-2">
          <div>• Record noises with the hood open and A/C, radio off.</div>
          <div>• Note if pitch changes with speed (wheels) or RPM (engine).</div>
          <div>• Keep your service history in the Logbook tab.</div>
        </CardContent>
      </Card>
    </div>
  );
}

function MechanicsTab() {
  const [coords, setCoords] = useState(null);
  const [favorites, setFavorites] = useLocalStorage(STORAGE.favorites, []);

  useEffect(() => {
    navigator.geolocation?.getCurrentPosition(
      (pos) => setCoords({ lat: pos.coords.latitude, lon: pos.coords.longitude }),
      () => setCoords(null),
      { enableHighAccuracy: true, timeout: 8000 }
    );
  }, []);

  const list = MOCK_MECHANICS.map((m) => ({
    ...m,
    distance: coords ? distanceMiles(coords, { lat: m.lat, lon: m.lon }) : null,
  })).sort((a, b) => (a.distance ?? 999) - (b.distance ?? 999));

  const toggleFav = (id) => {
    setFavorites((prev) => (prev.includes(id) ? prev.filter((x) => x !== id) : [...prev, id]));
  };

  return (
    <div className="grid gap-4 md:grid-cols-2">
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center gap-2"><MapPin className="h-5 w-5 text-orange-500" /> Nearby Mechanics</CardTitle>
        </CardHeader>
        <CardContent className="space-y-3">
          {list.map((m) => (
            <div key={m.id} className="rounded-xl border p-3 flex items-center justify-between">
              <div>
                <div className="font-medium">{m.name}</div>
                <div className="text-xs text-neutral-600">{m.phone} · {m.rating}★ {m.distance != null ? `· ${m.distance.toFixed(1)} mi` : ""}</div>
              </div>
              <Button variant={favorites.includes(m.id) ? "secondary" : "outline"} onClick={() => toggleFav(m.id)}>
                {favorites.includes(m.id) ? "Saved" : "Save"}
              </Button>
            </div>
          ))}
        </CardContent>
      </Card>

      <Card>
        <CardHeader>
          <CardTitle>Saved</CardTitle>
        </CardHeader>
        <CardContent className="space-y-3">
          {favorites.length === 0 && <div className="text-sm text-neutral-600">No favorites yet.</div>}
          {list.filter((m) => favorites.includes(m.id)).map((m) => (
            <div key={m.id} className="rounded-xl border p-3">
              <div className="font-medium">{m.name}</div>
              <div className="text-xs text-neutral-600">{m.phone}</div>
            </div>
          ))}
        </CardContent>
      </Card>
    </div>
  );
}

export default function AutoWhisperApp() {
  const [profile, setProfile] = useLocalStorage(STORAGE.profile, defaultProfile);
  const [settingsOpen, setSettingsOpen] = useState(false);

  return (
    <div className="min-h-screen bg-white text-neutral-900">
      <Header onOpenSettings={() => setSettingsOpen(true)} />
      <SettingsPanel open={settingsOpen} onClose={() => setSettingsOpen(false)} profile={profile} setProfile={setProfile} />

      <main className="mx-auto max-w-6xl px-4 py-6">
        <div className="mb-6">
          <div className="text-2xl font-bold">Welcome back{profile.owner ? ", " + profile.owner : ""}.</div>
          <div className="text-sm text-neutral-600">Car: {profile.vehicle || "Set your vehicle in Settings"}</div>
        </div>

        <Tabs defaultValue="diagnose" className="w-full">
          <TabsList className="grid w-full grid-cols-4">
            <TabsTrigger value="diagnose">Diagnose</TabsTrigger>
            <TabsTrigger value="logbook">Logbook</TabsTrigger>
            <TabsTrigger value="chat">Chat</TabsTrigger>
            <TabsTrigger value="mechanics">Mechanics</TabsTrigger>
          </TabsList>

          <TabsContent value="diagnose" className="mt-4"><DiagnoseTab /></TabsContent>
          <TabsContent value="logbook" className="mt-4"><LogbookTab /></TabsContent>
          <TabsContent value="chat" className="mt-4"><ChatTab profile={profile} /></TabsContent>
          <TabsContent value="mechanics" className="mt-4"><MechanicsTab /></TabsContent>
        </Tabs>
      </main>

      <footer className="mt-10 border-t">
        <div className="mx-auto max-w-6xl px-4 py-6 text-xs text-neutral-500 flex items-center justify-between">
          <div>© {new Date().getFullYear()} AutoWhisper. "When your car talks, AutoWhisper listens."</div>
          <div>Demo only — not a substitute for a certified mechanic.</div>
        </div>
      </footer>
    </div>
  );
}

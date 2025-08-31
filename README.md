# women-child-safety-app
# 🚨 Women & Child Safety App

A simple **prototype web application** built with HTML, CSS, and JavaScript to help women and children in emergency situations by providing quick SOS alerts, live location sharing, and trusted contact management.

---

## ✨ Features
- 📍 **Live Location** – Fetches current GPS coordinates.  
- 🚨 **SOS Button** – Opens WhatsApp/SMS with a pre-filled emergency message and location link.  
- 👥 **Trusted Contacts** – Add contacts and quickly send SOS messages to them.  
- 📤 **Quick Sharing** – Copy/share message instantly.  

---

## 📂 Project Structure
****

import React, { useEffect, useMemo, useRef, useState } from "react";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Switch } from "@/components/ui/switch";
import { Textarea } from "@/components/ui/textarea";
import { toast } from "sonner";
import { ShieldAlert, PhoneCall, Siren, Share2, MapPin, Plus, Trash2, Copy, Timer, MessageSquareShare } from "lucide-react";
import { motion } from "framer-motion";

// --- Small utilities ---
const fmtPhone = (s) => s.replace(/\D/g, "");
const mapLink = (lat, lng) => `https://maps.google.com/?q=${lat},${lng}`;
const defaultMsg = (lat, lng) =>
  `🚨 SOS! I need help.\nLocation: ${lat?.toFixed?.(5)}, ${lng?.toFixed?.(5)}
📍 ${mapLink(lat, lng)}\n`;

const useLocalState = (key, initial) => {
  const [v, setV] = useState(() => {
    try {
      const raw = localStorage.getItem(key);
      return raw ? JSON.parse(raw) : initial;
    } catch {
      return initial;
    }
  });
  useEffect(() => {
    try { localStorage.setItem(key, JSON.stringify(v)); } catch {}
  }, [key, v]);
  return [v, setV];
};

export default function SafetyApp() {
  const [coords, setCoords] = useState(null);
  const [watching, setWatching] = useState(false);
  const watchId = useRef(null);
  const [message, setMessage] = useState("");
  const [isSharing, setIsSharing] = useState(false);
  const [contacts, setContacts] = useLocalState("safety.contacts", []);
  const [name, setName] = useLocalState("user.name", "");
  const [incidents, setIncidents] = useLocalState("safety.incidents", []);
  const [fakeCallIn, setFakeCallIn] = useState(10);
  const [fakeCallActive, setFakeCallActive] = useState(false);
  const [sirenOn, setSirenOn] = useState(false);
  const audioCtxRef = useRef(null);
  const sirenNodesRef = useRef({});

  // Get one-time location immediately (if allowed)
  useEffect(() => {
    if (!navigator.geolocation) return;
    navigator.geolocation.getCurrentPosition(
      (pos) => setCoords({ lat: pos.coords.latitude, lng: pos.coords.longitude }),
      () => {},
      { enableHighAccuracy: true, timeout: 5000 }
    );
  }, []);

  // Build outgoing SOS text
  const sosText = useMemo(() => {
    if (!coords) return "🚨 SOS! I need help. (Location unavailable)";
    const base = defaultMsg(coords.lat, coords.lng);
    const nameLine = name ? `👤 ${name}\n` : "";
    const contactsLine = contacts?.length
      ? `📞 Notify: ${contacts.map((c) => c.name || c.phone).join(", ")}\n`
      : "";
    return `${nameLine}${base}${message ? `Note: ${message}\n` : ""}${contactsLine}`.trim();
  }, [coords, message, contacts, name]);

  // Live tracking via watchPosition
  const toggleWatch = (on) => {
    if (!navigator.geolocation) {
      toast.error("Geolocation not supported by this browser");
      return;
    }
    if (on) {
      const id = navigator.geolocation.watchPosition(
        (pos) => setCoords({ lat: pos.coords.latitude, lng: pos.coords.longitude }),
        (err) => toast.error(err.message || "Failed to watch location"),
        { enableHighAccuracy: true, maximumAge: 2000, timeout: 10000 }
      );
      watchId.current = id;
      setWatching(true);
    } else {
      try { navigator.geolocation.clearWatch(watchId.current); } catch {}
      watchId.current = null;
      setWatching(false);
    }
  };

  // WebAudio siren (no external assets)
  const toggleSiren = (on) => {
    setSirenOn(on);
    if (on) {
      if (!audioCtxRef.current) audioCtxRef.current = new (window.AudioContext || window.webkitAudioContext)();
      const ctx = audioCtxRef.current;
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      osc.type = "sawtooth";
      let t = 0;
      const base = 600;
      const sweep = () => {
        if (!sirenOn) return;
        osc.frequency.setValueAtTime(base + 400 * Math.abs(Math.sin(t)), ctx.currentTime);
        t += 0.08;
        sirenNodesRef.current.raf = requestAnimationFrame(sweep);
      };
      osc.connect(gain); gain.connect(ctx.destination);
      gain.gain.value = 0.05; // soft but audible
      osc.start();
      sirenNodesRef.current.osc = osc;
      sweep();
    } else {
      const { osc, raf } = sirenNodesRef.current;
      if (raf) cancelAnimationFrame(raf);
      try { osc?.stop(); } catch {}
      sirenNodesRef.current = {};
    }
  };

  // Fake call timer
  useEffect(() => {
    if (!fakeCallActive) return;
    let s = fakeCallIn;
    const id = setInterval(() => {
      s -= 1;
      setFakeCallIn(s);
      if (s <= 0) {
        clearInterval(id);
        // play a quick ringtone-like beeps
        const ctx = audioCtxRef.current || new (window.AudioContext || window.webkitAudioContext)();
        audioCtxRef.current = ctx;
        const osc = ctx.createOscillator();
        const gain = ctx.createGain();
        osc.type = "square"; osc.frequency.value = 880;
        osc.connect(gain); gain.connect(ctx.destination);
        gain.gain.value = 0.02;
        osc.start();
        setTimeout(() => { osc.stop(); toast.info("Incoming call… (simulated)"); setFakeCallActive(false); setFakeCallIn(10); }, 1200);
      }
    }, 1000);
    return () => clearInterval(id);
  }, [fakeCallActive]);

  // Share helpers (no backend needed)
  const shareSOS = async () => {
    if (!coords) {
      toast.error("Location not available yet. Please allow location access.");
      return;
    }
    const text = sosText;
    setIsSharing(true);
    try {
      if (navigator.share) {
        await navigator.share({ title: "SOS", text });
        toast.success("Shared via native share sheet");
      } else {
        // Fallback: open WhatsApp (web) prefilled
        const wa = `https://wa.me/?text=${encodeURIComponent(text)}`;
        window.open(wa, "_blank");
      }
    } catch (e) {
      toast.error("Share canceled or failed");
    } finally {
      setIsSharing(false);
    }
  };

  const addContact = (name, phone) => {
    phone = fmtPhone(phone);
    if (!phone) return toast.error("Enter a valid phone number");
    setContacts([...(contacts || []), { name: name || phone, phone }]);
  };
  const removeContact = (idx) => {
    const next = [...contacts];
    next.splice(idx, 1);
    setContacts(next);
  };

  const copyText = async (t) => {
    try { await navigator.clipboard.writeText(t); toast.success("Copied to clipboard"); } catch {}
  };

  const logIncident = (e) => {
    e.preventDefault();
    const form = new FormData(e.target);
    const entry = {
      when: new Date().toISOString(),
      where: form.get("where") || (coords ? `${coords.lat},${coords.lng}` : "unknown"),
      details: form.get("details") || "",
    };
    setIncidents([entry, ...incidents]);
    (e.target).reset();
    toast.success("Saved locally (private)");
  };

  return (
    <div className="min-h-screen bg-gradient-to-b from-rose-50 to-rose-100 p-4 sm:p-8">
      <div className="max-w-5xl mx-auto space-y-6">
        <header className="flex items-center justify-between">
          <div>
            <h1 className="text-3xl sm:text-4xl font-bold tracking-tight">Women & Child Safety</h1>
            <p className="text-sm text-muted-foreground">Quick SOS • Live location • Trusted contacts • Siren • Fake call</p>
          </div>
          <div className="flex items-center gap-2">
            <Label htmlFor="name" className="text-xs">Your name</Label>
            <Input id="name" value={name} onChange={(e)=>setName(e.target.value)} placeholder="Optional" className="w-40"/>
          </div>
        </header>

        {/* SOS Card */}
        <Card className="shadow-xl">
          <CardHeader className="pb-2">
            <CardTitle className="flex items-center gap-2"><ShieldAlert className="w-5 h-5"/> Emergency SOS</CardTitle>
          </CardHeader>
          <CardContent className="space-y-4">
            <div className="grid grid-cols-1 md:grid-cols-3 gap-4 items-center">
              <motion.button
                whileTap={{ scale: 0.95 }}
                onClick={shareSOS}
                className={`md:col-span-1 rounded-full aspect-square w-48 h-48 mx-auto flex items-center justify-center text-white text-3xl font-extrabold shadow-2xl ${isSharing ? "bg-zinc-400" : "bg-rose-600"}`}
              >
                SOS
              </motion.button>

              <div className="md:col-span-2 space-y-3">
                <div className="flex items-center gap-2 text-sm"><MapPin className="w-4 h-4"/>
                  {coords ? (
                    <span>
                      {coords.lat.toFixed(5)}, {coords.lng.toFixed(5)} — <a className="underline" href={mapLink(coords.lat, coords.lng)} target="_blank" rel="noreferrer">Open map</a>
                    </span>
                  ) : (
                    <span>Waiting for location permission…</span>
                  )}
                </div>
                <div className="flex items-center gap-3">
                  <Switch checked={watching} onCheckedChange={toggleWatch} id="watch"/>
                  <Label htmlFor="watch">Live tracking</Label>
                </div>
                <div className="grid grid-cols-1 sm:grid-cols-2 gap-2">
                  <Button variant="outline" onClick={() => copyText(sosText)} className="gap-2"><Copy className="w-4 h-4"/> Copy SOS text</Button>
                  <Button variant="secondary" onClick={shareSOS} className="gap-2"><Share2 className="w-4 h-4"/> Share now</Button>
                </div>
                <Textarea placeholder="Add a note (what happened?)" value={message} onChange={(e)=>setMessage(e.target.value)}/>
                <div className="text-xs text-muted-foreground">Tip: If native share is unavailable, we will open WhatsApp with a pre‑filled message. You can also use your SMS app via <code>sms:</code> links.</div>
              </div>
            </div>
          </CardContent>
        </Card>

        {/* Contacts */}
        <Card>
          <CardHeader className="pb-2"><CardTitle className="flex items-center gap-2"><MessageSquareShare className="w-5 h-5"/> Trusted contacts</CardTitle></CardHeader>
          <CardContent className="space-y-3">
            <div className="grid grid-cols-1 sm:grid-cols-3 gap-2">
              <Input placeholder="Name (optional)" id="cname"/>
              <Input placeholder="Phone (digits only)" id="cphone"/>
              <Button onClick={() => {
                const n = document.getElementById("cname").value;
                const p = document.getElementById("cphone").value;
                addContact(n, p);
                document.getElementById("cname").value = "";
                document.getElementById("cphone").value = "";
              }} className="gap-2"><Plus className="w-4 h-4"/> Add</Button>
            </div>
            <div className="space-y-2">
              {(contacts||[]).length === 0 && <div className="text-sm text-muted-foreground">No contacts added yet.</div>}
              {(contacts||[]).map((c, i) => (
                <div key={i} className="flex items-center justify-between bg-white rounded-xl p-2 shadow border">
                  <div className="text-sm">
                    <div className="font-medium">{c.name || c.phone}</div>
                    <div className="text-xs text-muted-foreground">{c.phone}</div>
                  </div>
                  <div className="flex items-center gap-2">
                    <a className="underline text-xs" href={`sms:${c.phone}?&body=${encodeURIComponent(sosText)}`}>SMS</a>
                    <a className="underline text-xs" href={`https://wa.me/${c.phone}?text=${encodeURIComponent(sosText)}`} target="_blank" rel="noreferrer">WhatsApp</a>
                    <Button variant="ghost" size="icon" onClick={() => removeContact(i)}><Trash2 className="w-4 h-4"/></Button>
                  </div>
                </div>
              ))}
            </div>
          </CardContent>
        </Card>

        {/* Siren & Fake Call */}
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <Card>
            <CardHeader className="pb-2"><CardTitle className="flex items-center gap-2"><Siren className="w-5 h-5"/> Panic siren</CardTitle></CardHeader>
            <CardContent className="space-y-3">
              <div className="flex items-center gap-3">
                <Switch checked={sirenOn} onCheckedChange={toggleSiren} id="siren"/>
                <Label htmlFor="siren">Play loud siren</Label>
              </div>
              <div className="text-xs text-muted-foreground">Uses your device speaker to draw attention. Remember to turn volume up.</div>
            </CardContent>
          </Card>

          <Card>
            <CardHeader className="pb-2"><CardTitle className="flex items-center gap-2"><PhoneCall className="w-5 h-5"/> Fake call</CardTitle></CardHeader>
            <CardContent className="space-y-3">
              <div className="flex items-center gap-3">
                <Label htmlFor="timer" className="w-24">In (sec)</Label>
                <Input id="timer" type="number" min={3} max={60} value={fakeCallIn} onChange={(e)=>setFakeCallIn(parseInt(e.target.value || 10))}/>
                <Button onClick={()=> setFakeCallActive(true)} className="gap-2"><Timer className="w-4 h-4"/> Start</Button>
              </div>
              {fakeCallActive && <div className="text-sm">Incoming call will ring in <b>{fakeCallIn}s</b>…</div>}
              <div className="text-xs text-muted-foreground">Creates a quick distraction or exit strategy.</div>
            </CardContent>
          </Card>
        </div>

        {/* Quick report (local) */}
        <Card>
          <CardHeader className="pb-2"><CardTitle>Quick incident note (private)</CardTitle></CardHeader>
          <CardContent>
            <form onSubmit={logIncident} className="space-y-3">
              <div className="grid grid-cols-1 sm:grid-cols-2 gap-3">
                <div>
                  <Label htmlFor="where">Where</Label>
                  <Input id="where" name="where" placeholder="Address or auto (uses GPS if empty)"/>
                </div>
                <div>
                  <Label htmlFor="details">What happened</Label>
                  <Textarea id="details" name="details" placeholder="Short description"/>
                </div>
              </div>
              <Button type="submit" variant="outline">Save locally</Button>
            </form>
            {incidents.length > 0 && (
              <div className="mt-4 space-y-2">
                <div className="text-sm font-medium">Saved notes</div>
                {incidents.map((it, idx) => (
                  <div key={idx} className="text-xs p-2 rounded border bg-white">
                    <div><b>When:</b> {new Date(it.when).toLocaleString()}</div>
                    <div><b>Where:</b> {it.where}</div>
                    <div><b>Details:</b> {it.details}</div>
                  </div>
                ))}
              </div>
            )}
          </CardContent>
        </Card>

        {/* Safety tips */}
        <Card>
          <CardHeader className="pb-2"><CardTitle>Quick Safety Tips</CardTitle></CardHeader>
          <CardContent>
            <ul className="list-disc pl-6 text-sm space-y-1 text-muted-foreground">
              <li>Share live location with a trusted contact when commuting alone.</li>
              <li>Keep your phone charged; enable emergency SOS on your device settings.</li>
              <li>Avoid sharing precise home location publicly.</li>
              <li>Trust your instincts; leave situations that feel unsafe.</li>
              <li>In India, dial <b>112</b> for emergency services; for women’s helpline dial <b>1091</b>.</li>
            </ul>
          </CardContent>
        </Card>

        <footer className="text-center text-xs text-muted-foreground pt-6">
          <div>Prototype only. Not a substitute for professional emergency services.</div>
        </footer>
      </div>
    </div>
  );
}

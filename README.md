# Music-Vibes-
Listen to your favourite songs 
import React, { useEffect, useMemo, useRef, useState } from "react";
import { motion } from "framer-motion";
import {
  Play,
  Pause,
  SkipBack,
  SkipForward,
  Shuffle,
  Repeat,
  Volume2,
  VolumeX,
  Upload,
  PlusCircle,
  FolderPlus,
  Search,
  Music2,
  Heart,
  ListMusic,
  Download,
  Trash2,
  Edit3,
  Save,
  X,
} from "lucide-react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Slider } from "@/components/ui/slider";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

// ---------- Utility helpers ----------
const fmtTime = (secs) => {
  if (!Number.isFinite(secs)) return "0:00";
  const s = Math.floor(secs % 60)
    .toString()
    .padStart(2, "0");
  const m = Math.floor(secs / 60);
  return `${m}:${s}`;
};

const uid = () => Math.random().toString(36).slice(2);

// ---------- Main App ----------
export default function MusicApp() {
  const audioRef = useRef(null);
  const canvasRef = useRef(null);
  const [tracks, setTracks] = useState(() => {
    // Seed with a few royalty-free demo tracks
    // You can replace these with your own URLs.
    return [
      {
        id: uid(),
        title: "Dreams of Rivers",
        artist: "Komiku",
        src: "https://files.freemusicarchive.org/storage-freemusicarchive-org/music/CC-BY/Komiku/It_is_time_for_adventure/Komiku_-_14_-_Dreams_of_Rivers.mp3",
        artwork: "https://picsum.photos/seed/river/200/200",
        duration: 0,
        liked: false,
      },
      {
        id: uid(),
        title: "Space Jazz",
        artist: "Kevin MacLeod",
        src: "https://incompetech.com/music/royalty-free/mp3-royaltyfree/Space%20Jazz.mp3",
        artwork: "https://picsum.photos/seed/jazz/200/200",
        duration: 0,
        liked: false,
      },
      {
        id: uid(),
        title: "City Night Ambient",
        artist: "Monplaisir",
        src: "https://files.freemusicarchive.org/storage-freemusicarchive-org/music/no_curator/Monplaisir/Ambient_Glass/Monplaisir_-_05_-_City_Night_Ambient.mp3",
        artwork: "https://picsum.photos/seed/city/200/200",
        duration: 0,
        liked: false,
      },
    ];
  });
  const [query, setQuery] = useState("");
  const [currentId, setCurrentId] = useState(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [progress, setProgress] = useState(0);
  const [duration, setDuration] = useState(0);
  const [volume, setVolume] = useState(() => {
    const v = localStorage.getItem("musicapp.volume");
    return v ? Number(v) : 0.8;
  });
  const [muted, setMuted] = useState(false);
  const [repeat, setRepeat] = useState(false);
  const [shuffle, setShuffle] = useState(false);
  const [queue, setQueue] = useState(() => []);
  const [playlists, setPlaylists] = useState(() => {
    const saved = localStorage.getItem("musicapp.playlists");
    return saved ? JSON.parse(saved) : { Favorites: [] };
  });
  const [activeList, setActiveList] = useState("All Songs");
  const [editingPlaylist, setEditingPlaylist] = useState(null);
  const [newPlaylistName, setNewPlaylistName] = useState("");

  // Derived
  const currentIndex = useMemo(
    () => queue.findIndex((id) => id === currentId),
    [queue, currentId]
  );

  const nowPlaying = useMemo(
    () => tracks.find((t) => t.id === currentId) || null,
    [tracks, currentId]
  );

  const visibleTracks = useMemo(() => {
    let list = tracks;
    if (activeList !== "All Songs") {
      const ids = new Set(playlists[activeList] || []);
      list = tracks.filter((t) => ids.has(t.id));
    }
    if (query.trim()) {
      const q = query.toLowerCase();
      list = list.filter(
        (t) =>
          t.title.toLowerCase().includes(q) ||
          t.artist.toLowerCase().includes(q)
      );
    }
    return list;
  }, [tracks, query, activeList, playlists]);

  // Persist playlists + volume
  useEffect(() => {
    localStorage.setItem("musicapp.playlists", JSON.stringify(playlists));
  }, [playlists]);
  useEffect(() => {
    localStorage.setItem("musicapp.volume", String(volume));
  }, [volume]);

  // Setup audio events
  useEffect(() => {
    const audio = audioRef.current;
    if (!audio) return;
    audio.volume = muted ? 0 : volume;

    const onTime = () => setProgress(audio.currentTime);
    const onLoaded = () => setDuration(audio.duration || 0);
    const onEnd = () => {
      if (repeat) {
        audio.currentTime = 0;
        audio.play();
      } else {
        handleNext();
      }
    };

    audio.addEventListener("timeupdate", onTime);
    audio.addEventListener("loadedmetadata", onLoaded);
    audio.addEventListener("ended", onEnd);
    return () => {
      audio.removeEventListener("timeupdate", onTime);
      audio.removeEventListener("loadedmetadata", onLoaded);
      audio.removeEventListener("ended", onEnd);
    };
  }, [repeat, volume, muted]);

  // Load durations lazily for seed tracks
  useEffect(() => {
    const audio = new Audio();
    let isCancelled = false;
    const load = async (t) =>
      new Promise((res) => {
        audio.src = t.src;
        audio.addEventListener("loadedmetadata", () => res(audio.duration), {
          once: true,
        });
        audio.load();
      });
    (async () => {
      const next = await Promise.all(
        tracks.map(async (t) => ({ ...t, duration: t.duration || (await load(t)) }))
      );
      if (!isCancelled) setTracks(next);
    })();
    return () => {
      isCancelled = true;
      audio.src = "";
    };
  }, []);

  // Visualizer via Web Audio API
  useEffect(() => {
    if (!audioRef.current || !canvasRef.current) return;
    const audio = audioRef.current;
    const ctx = new (window.AudioContext || window.webkitAudioContext)();
    const source = ctx.createMediaElementSource(audio);
    const analyser = ctx.createAnalyser();
    analyser.fftSize = 256;
    source.connect(analyser);
    analyser.connect(ctx.destination);

    const canvas = canvasRef.current;
    const cctx = canvas.getContext("2d");
    const bufferLength = analyser.frequencyBinCount;
    const dataArray = new Uint8Array(bufferLength);

    let rafId;
    const draw = () => {
      analyser.getByteFrequencyData(dataArray);
      cctx.clearRect(0, 0, canvas.width, canvas.height);
      const barWidth = (canvas.width / bufferLength) * 1.5;
      let x = 0;
      for (let i = 0; i < bufferLength; i++) {
        const v = dataArray[i] / 255;
        const h = v * canvas.height;
        cctx.fillStyle = `rgba(0,0,0,${0.6 + v * 0.4})`;
        cctx.fillRect(x, canvas.height - h, barWidth - 2, h);
        x += barWidth;
      }
      rafId = requestAnimationFrame(draw);
    };
    draw();
    return () => cancelAnimationFrame(rafId);
  }, [currentId]);

  // Keyboard shortcuts
  useEffect(() => {
    const onKey = (e) => {
      if (e.target.closest("input, textarea")) return;
      if (e.code === "Space") {
        e.preventDefault();
        togglePlay();
      } else if (e.code === "ArrowRight") {
        seek(Math.min(duration - 1, progress + 5));
      } else if (e.code === "ArrowLeft") {
        seek(Math.max(0, progress - 5));
      } else if (e.code === "ArrowUp") {
        setVolume((v) => Math.min(1, v + 0.05));
      } else if (e.code === "ArrowDown") {
        setVolume((v) => Math.max(0, v - 0.05));
      }
    };
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [progress, duration]);

  // ---------- Actions ----------
  const startQueueFrom = (id, list) => {
    const ids = (list || visibleTracks).map((t) => t.id);
    const startIdx = ids.indexOf(id);
    const ordered = [...ids.slice(startIdx), ...ids.slice(0, startIdx)];
    setQueue(ordered);
    setCurrentId(id);
    setTimeout(() => play(), 0);
  };

  const togglePlay = () => (isPlaying ? pause() : play());
  const play = () => {
    audioRef.current?.play();
    setIsPlaying(true);
  };
  const pause = () => {
    audioRef.current?.pause();
    setIsPlaying(false);
  };
  const handlePrev = () => {
    if (progress > 3) {
      seek(0);
      return;
    }
    if (shuffle) {
      const rest = queue.filter((id) => id !== currentId);
      const nextId = rest[Math.floor(Math.random() * rest.length)] || currentId;
      setCurrentId(nextId);
      return;
    }
    const idx = currentIndex;
    const prevId = queue[(idx - 1 + queue.length) % queue.length];
    setCurrentId(prevId);
  };
  const handleNext = () => {
    if (shuffle) {
      const rest = queue.filter((id) => id !== currentId);
      const nextId = rest[Math.floor(Math.random() * rest.length)] || currentId;
      setCurrentId(nextId);
      return;
    }
    const idx = currentIndex;
    const nextId = queue[(idx + 1) % queue.length];
    setCurrentId(nextId);
  };
  const seek = (t) => {
    const audio = audioRef.current;
    if (!audio) return;
    audio.currentTime = t;
    setProgress(t);
  };

  const addLocalFiles = async (files) => {
    const added = Array.from(files)
      .filter((f) => f.type.startsWith("audio/"))
      .map((f) => ({
        id: uid(),
        title: f.name.replace(/\.[^.]+$/, ""),
        artist: "Local File",
        src: URL.createObjectURL(f),
        artwork: "https://picsum.photos/seed/cover/200/200",
        duration: 0,
        liked: false,
      }));
    setTracks((cur) => [...added, ...cur]);
  };

  const toggleLike = (id) => {
    setTracks((ts) =>
      ts.map((t) => (t.id === id ? { ...t, liked: !t.liked } : t))
    );
    setPlaylists((p) => {
      const setFav = new Set(p.Favorites || []);
      if (setFav.has(id)) setFav.delete(id);
      else setFav.add(id);
      return { ...p, Favorites: Array.from(setFav) };
    });
  };

  const createPlaylist = (name) => {
    if (!name.trim() || playlists[name]) return;
    setPlaylists({ ...playlists, [name]: [] });
    setActiveList(name);
  };
  const deletePlaylist = (name) => {
    if (name === "Favorites" || name === "All Songs") return;
    const { [name]: _, ...rest } = playlists;
    setPlaylists(rest);
    setActiveList("All Songs");
  };
  const renamePlaylist = (oldName, newName) => {
    if (!playlists[oldName] || playlists[newName]) return;
    const entries = Object.entries(playlists).map(([k, v]) => [
      k === oldName ? newName : k,
      v,
    ]);
    setPlaylists(Object.fromEntries(entries));
    if (activeList === oldName) setActiveList(newName);
  };
  const addToPlaylist = (name, id) => {
    setPlaylists((p) => {
      const setIds = new Set(p[name] || []);
      setIds.add(id);
      return { ...p, [name]: Array.from(setIds) };
    });
  };
  const removeFromPlaylist = (name, id) => {
    setPlaylists((p) => ({
      ...p,
      [name]: (p[name] || []).filter((x) => x !== id),
    }));
  };

  const moveInQueue = (from, to) => {
    setQueue((q) => {
      const next = [...q];
      const [item] = next.splice(from, 1);
      next.splice(to, 0, item);
      return next;
    });
  };

  // ---------- UI ----------
  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 text-slate-900">
      {/* Top bar */}
      <header className="sticky top-0 z-40 backdrop-blur bg-white/70 border-b border-slate-200">
        <div className="max-w-7xl mx-auto px-4 py-3 flex items-center gap-3">
          <Music2 className="w-6 h-6" />
          <h1 className="text-xl font-semibold">Waveflow</h1>
          <div className="ml-auto flex items-center gap-2 w-full max-w-xl">
            <div className="relative flex-1">
              <Search className="w-4 h-4 absolute left-3 top-1/2 -translate-y-1/2" />
              <Input
                placeholder="Search songs or artists…"
                className="pl-9"
                value={query}
                onChange={(e) => setQuery(e.target.value)}
              />
            </div>
            <label className="cursor-pointer inline-flex items-center gap-2 text-sm px-3 py-2 rounded-2xl bg-slate-900 text-white hover:opacity-90">
              <Upload className="w-4 h-4" />
              <span>Upload</span>
              <input
                type="file"
                accept="audio/*"
                multiple
                className="hidden"
                onChange={(e) => addLocalFiles(e.target.files)}
              />
            </label>
          </div>
        </div>
      </header>

      {/* Grid layout */}
      <main className="max-w-7xl mx-auto p-4 grid grid-cols-12 gap-4">
        {/* Left: Playlists */}
        <aside className="col-span-12 md:col-span-3 space-y-3">
          <Card className="rounded-2xl shadow-sm">
            <CardHeader className="pb-2">
              <CardTitle className="text-base">Playlists</CardTitle>
            </CardHeader>
            <CardContent className="space-y-2">
              {(["All Songs", ...Object.keys(playlists)]).map((name) => (
                <div
                  key={name}
                  className={`flex items-center justify-between px-3 py-2 rounded-xl cursor-pointer ${
                    activeList === name ? "bg-slate-100" : "hover:bg-slate-50"
                  }`}
                  onClick={() => setActiveList(name)}
                >
                  <div className="flex items-center gap-2">
                    <ListMusic className="w-4 h-4" />
                    <span className="text-sm">{name}</span>
                  </div>
                  {name !== "All Songs" && name !== "Favorites" && (
                    <div className="flex items-center gap-1 opacity-70">
                      {editingPlaylist === name ? (
                        <>
                          <Button
                            size="icon"
                            variant="ghost"
                            onClick={(e) => {
                              e.stopPropagation();
                              const newName = newPlaylistName.trim();
                              if (newName) renamePlaylist(name, newName);
                              setEditingPlaylist(null);
                              setNewPlaylistName("");
                            }}
                          >
                            <Save className="w-4 h-4" />
                          </Button>
                          <Button
                            size="icon"
                            variant="ghost"
                            onClick={(e) => {
                              e.stopPropagation();
                              setEditingPlaylist(null);
                              setNewPlaylistName("");
                            }}
                          >
                            <X className="w-4 h-4" />
                          </Button>
                        </>
                      ) : (
                        <>
                          <Button
                            size="icon"
                            variant="ghost"
                            onClick={(e) => {
                              e.stopPropagation();
                              setEditingPlaylist(name);
                              setNewPlaylistName(name);
                            }}
                          >
                            <Edit3 className="w-4 h-4" />
                          </Button>
                          <Button
                            size="icon"
                            variant="ghost"
                            onClick={(e) => {
                              e.stopPropagation();
                              deletePlaylist(name);
                            }}
                          >
                            <Trash2 className="w-4 h-4" />
                          </Button>
                        </>
                      )}
                </div>
              ))}

              {editingPlaylist && (
                <div className="px-3">
                  <Input
                    value={newPlaylistName}
                    onChange={(e) => setNewPlaylistName(e.target.value)}
                    className="text-sm"
                  />
                </div>
              )}

              <div className="pt-2">
                <Button
                  className="w-full justify-start gap-2 rounded-xl"
                  variant="secondary"
                  onClick={() => {
                    const name = prompt("New playlist name?");
                    if (name) createPlaylist(name);
                  }}
                >
                  <FolderPlus className="w-4 h-4" /> New playlist
                </Button>
              </div>
            </CardContent>
          </Card>

          <Card className="rounded-2xl shadow-sm">
            <CardHeader className="pb-2">
              <CardTitle className="text-base">Queue</CardTitle>
            </CardHeader>
            <CardContent className="space-y-1 max-h-64 overflow-auto pr-1">
              {queue.length === 0 && (
                <p className="text-sm text-slate-500">Nothing queued yet</p>
              )}
              {queue.map((id, i) => {
                const t = tracks.find((x) => x.id === id);
                if (!t) return null;
                const isCur = id === currentId;
                return (
                  <div
                    key={id}
                    className={`flex items-center justify-between gap-2 px-2 py-1 rounded-lg ${
                      isCur ? "bg-slate-100" : "hover:bg-slate-50"
                    }`}
                  >
                    <div className="truncate">
                      <div className="text-sm font-medium truncate">{t.title}</div>
                      <div className="text-xs text-slate-500 truncate">{t.artist}</div>
                    </div>
                    <div className="flex items-center gap-1">
                      <Button
                        size="icon"
                        variant="ghost"
                        onClick={() => moveInQueue(i, Math.max(0, i - 1))}
                        title="Move up"
                      >
                        <SkipBack className="w-4 h-4" />
                      </Button>
                      <Button
                        size="icon"
                        variant="ghost"
                        onClick={() => moveInQueue(i, Math.min(queue.length - 1, i + 1))}
                        title="Move down"
                      >
                        <SkipForward className="w-4 h-4" />
                      </Button>
                    </div>
                  </div>
                );
              })}
            </CardContent>
          </Card>
        </aside>

        {/* Center: Library */}
        <section className="col-span-12 md:col-span-6 space-y-3">
          <Card className="rounded-2xl shadow-sm">
            <CardHeader className="pb-2">
              <CardTitle className="text-base">Library</CardTitle>
            </CardHeader>
            <CardContent className="space-y-2">
              {visibleTracks.map((t) => (
                <motion.div
                  layout
                  key={t.id}
                  className="grid grid-cols-12 gap-3 items-center p-2 rounded-xl hover:bg-slate-50"
                >
                  <img
                    src={t.artwork}
                    alt="art"
                    className="col-span-2 w-14 h-14 object-cover rounded-xl"
                  />
                  <div className="col-span-6 truncate">
                    <div className="font-medium truncate">{t.title}</div>
                    <div className="text-sm text-slate-500 truncate">{t.artist}</div>
                  </div>
                  <div className="col-span-2 text-sm text-slate-500 text-right">
                    {fmtTime(t.duration)}
                  </div>
                  <div className="col-span-2 flex items-center justify-end gap-1">
                    <Button
                      size="icon"
                      variant="ghost"

# Hand Cricket Source Code

Here is the complete codebase organized by folder structure. You can use this to review all the code at a glance!

## Root Configuration Files

### `package.json`
```json
{
  "name": "hand-cricket",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "seed": "node scripts/seed.js"
  },
  "dependencies": {
    "firebase": "^10.12.0",
    "firebase-admin": "^12.1.0",
    "react": "^18",
    "react-dom": "^18",
    "next": "14.2.3",
    "framer-motion": "^11.2.6",
    "lucide-react": "^0.378.0",
    "tailwindcss": "^3.4.1",
    "postcss": "^8.4.38",
    "autoprefixer": "^10.4.19"
  }
}
```

### `tailwind.config.js`
```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./src/pages/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/components/**/*.{js,ts,jsx,tsx,mdx}",
    "./src/app/**/*.{js,ts,jsx,tsx,mdx}",
  ],
  theme: {
    extend: {
      colors: {
        cricket: {
          red: '#D32F2F',
          darkRed: '#B71C1C',
          black: '#121212',
          dark: '#1E1E1E',
          gold: '#FFD700',
          lightGold: '#FFEA00'
        }
      },
      backgroundImage: {
        "gradient-radial": "radial-gradient(var(--tw-gradient-stops))",
        "gradient-conic":
          "conic-gradient(from 180deg at 50% 50%, var(--tw-gradient-stops))",
      },
    },
  },
  plugins: [],
};
```

### `jsconfig.json`
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

---

## `src/lib/` (Helpers & Database)

### `src/lib/firebase.js`
```javascript
import { initializeApp, getApps } from "firebase/app";
import { getDatabase } from "firebase/database";
import { getFirestore } from "firebase/firestore";

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY || "dummy-api-key",
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN || "dummy-auth-domain",
  databaseURL: process.env.NEXT_PUBLIC_FIREBASE_DATABASE_URL || "https://dummy-db.firebaseio.com",
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID || "dummy-project-id",
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET || "dummy-storage-bucket",
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID || "dummy-sender-id",
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID || "dummy-app-id"
};

const app = getApps().length === 0 ? initializeApp(firebaseConfig) : getApps()[0];

export const rtdb = getDatabase(app);
export const db = getFirestore(app);

export default app;
```

### `src/lib/userService.js`
```javascript
import { doc, getDoc, setDoc, updateDoc, increment } from "firebase/firestore";
import { db } from "./firebase";

export const getLocalUserId = () => {
  if (typeof window === "undefined") return null;
  let userId = localStorage.getItem("handCricketUserId");
  if (!userId) {
    userId = `user_${Math.random().toString(36).substr(2, 9)}`;
    localStorage.setItem("handCricketUserId", userId);
  }
  return userId;
};

export const getLocalUserName = () => {
  if (typeof window === "undefined") return null;
  return localStorage.getItem("handCricketUserName") || "Anonymous";
};

export const setLocalUserName = (name) => {
  if (typeof window === "undefined") return;
  localStorage.setItem("handCricketUserName", name);
};

export const updatePlayerStats = async (runs, wickets, isWin) => {
  const userId = getLocalUserId();
  const userName = getLocalUserName();
  if (!userId) return;

  const userRef = doc(db, "leaderboard", userId);
  const userSnap = await getDoc(userRef);

  if (userSnap.exists()) {
    await updateDoc(userRef, {
      name: userName,
      runs: increment(runs),
      wickets: increment(wickets),
      wins: isWin ? increment(1) : increment(0)
    });
  } else {
    await setDoc(userRef, {
      name: userName,
      runs: runs,
      wickets: wickets,
      wins: isWin ? 1 : 0
    });
  }
};
```

---

## `src/hooks/` (Custom Hooks)

### `src/hooks/useMatchSync.js`
```javascript
import { useState, useEffect } from "react";
import { doc, onSnapshot, updateDoc } from "firebase/firestore";
import { db } from "@/lib/firebase";

export function useMatchSync(roomId, playerNum) {
  const [gameState, setGameState] = useState(null);
  const [error, setError] = useState("");

  useEffect(() => {
    if (!roomId) return;
    const roomRef = doc(db, "matches", roomId);
    const unsubscribe = onSnapshot(roomRef, (snapshot) => {
      if (snapshot.exists()) {
        const data = snapshot.data();
        setGameState(data);
        
        if ((data.status === "INNINGS_1" || data.status === "INNINGS_2") && 
            data.p1?.choice !== null && data.p2?.choice !== null &&
            data.p1?.choice !== undefined && data.p2?.choice !== undefined) {
          if (playerNum === 1) evaluateTurn(data, roomRef);
        }
      } else {
        setError("Room closed.");
        setGameState(null);
      }
    }, (err) => {
      console.error(err);
      setError("Failed to sync room data.");
    });
    return () => unsubscribe();
  }, [roomId, playerNum]);

  const evaluateTurn = async (data, roomRef) => {
    const choice1 = data.p1.choice;
    const choice2 = data.p2.choice;
    const batPlayer = data.p1.role === "BAT" ? "p1" : "p2";
    const bowlPlayer = data.p1.role === "BOWL" ? "p1" : "p2";
    const isWicket = choice1 === choice2;
    
    let updates = { "p1.choice": null, "p2.choice": null, turnId: (data.turnId || 1) + 1 };

    if (isWicket) {
      if (data.status === "INNINGS_1") {
        updates.status = "INNINGS_BREAK";
        updates.target = data[batPlayer].score + 1;
        updates[`${bowlPlayer}.wickets`] = (data[bowlPlayer].wickets || 0) + 1;
      } else if (data.status === "INNINGS_2") {
        updates.status = "GAME_OVER";
        updates.winner = bowlPlayer;
        updates[`${bowlPlayer}.wickets`] = (data[bowlPlayer].wickets || 0) + 1;
      }
    } else {
      const runsScored = data[batPlayer].choice;
      const newScore = data[batPlayer].score + runsScored;
      updates[`${batPlayer}.score`] = newScore;
      if (data.status === "INNINGS_2" && data.target && newScore >= data.target) {
        updates.status = "GAME_OVER";
        updates.winner = batPlayer;
      }
    }

    updates.isOut = isWicket;
    await updateDoc(roomRef, updates);
    if (isWicket) setTimeout(async () => { await updateDoc(roomRef, { isOut: false }); }, 2000);
  };

  const handleInput = async (num) => {
    if (!gameState || ["WAITING", "INNINGS_BREAK", "GAME_OVER"].includes(gameState.status) || gameState.isOut) return;
    const myPlayerKey = `p${playerNum}`;
    if (gameState[myPlayerKey]?.choice !== null) return;
    await updateDoc(doc(db, "matches", roomId), { [`${myPlayerKey}.choice`]: num });
  };

  const startSecondInnings = async () => {
    if (playerNum !== 1) return;
    await updateDoc(doc(db, "matches", roomId), {
      status: "INNINGS_2",
      "p1.role": gameState.p1.role === "BAT" ? "BOWL" : "BAT",
      "p2.role": gameState.p2.role === "BAT" ? "BOWL" : "BAT",
      "p1.choice": null,
      "p2.choice": null,
    });
  };

  const callToss = async (call) => {
    const flip = Math.random() > 0.5 ? "HEADS" : "TAILS";
    const p1Wins = call === flip;
    await updateDoc(doc(db, "matches", roomId), {
      status: p1Wins ? "TOSS_CHOICE_P1" : "TOSS_CHOICE_P2",
      tossWinner: p1Wins ? "p1" : "p2",
      tossResult: flip
    });
  };

  const chooseBatBowl = async (choice) => {
    const isP1 = gameState.tossWinner === "p1";
    const p1Role = isP1 ? choice : (choice === "BAT" ? "BOWL" : "BAT");
    const p2Role = p1Role === "BAT" ? "BOWL" : "BAT";
    await updateDoc(doc(db, "matches", roomId), {
      status: "INNINGS_1", "p1.role": p1Role, "p2.role": p2Role
    });
  };

  return { gameState, error, handleInput, startSecondInnings, callToss, chooseBatBowl };
}
```

---

## `src/app/` (Application Pages)

### `src/app/page.jsx` (Splash Screen)
```javascript
"use client";

import { motion } from "framer-motion";
import { Play, Users, Trophy } from "lucide-react";
import Link from "next/link";
import { useState, useEffect } from "react";
import { getLocalUserName, setLocalUserName } from "@/lib/userService";

export default function Home() {
  const [name, setName] = useState("");
  const [hasName, setHasName] = useState(false);

  useEffect(() => {
    const savedName = getLocalUserName();
    if (savedName && savedName !== "Anonymous") {
      setHasName(true);
      setName(savedName);
    }
  }, []);

  const saveName = (e) => {
    e.preventDefault();
    if (name.trim()) {
      setLocalUserName(name.trim());
      setHasName(true);
    }
  };

  return (
    <main className="flex min-h-screen flex-col items-center justify-center p-6 bg-cricket-black overflow-hidden relative">
      <div className="absolute top-0 left-0 w-full h-full overflow-hidden z-0 pointer-events-none">
        <div className="absolute -top-40 -left-40 w-96 h-96 bg-cricket-darkRed rounded-full mix-blend-multiply filter blur-3xl opacity-30 animate-pulse"></div>
        <div className="absolute top-40 -right-40 w-96 h-96 bg-cricket-red rounded-full mix-blend-multiply filter blur-3xl opacity-20 animate-pulse" style={{ animationDelay: "2s" }}></div>
        <div className="absolute -bottom-40 left-20 w-96 h-96 bg-cricket-gold rounded-full mix-blend-multiply filter blur-3xl opacity-10 animate-pulse" style={{ animationDelay: "4s" }}></div>
      </div>

      <div className="z-10 flex flex-col items-center max-w-md w-full gap-10">
        <motion.div initial={{ y: -50, opacity: 0 }} animate={{ y: 0, opacity: 1 }} transition={{ duration: 0.8, ease: "easeOut" }} className="text-center">
          <h1 className="text-6xl font-black italic tracking-tighter text-transparent bg-clip-text bg-gradient-to-br from-white via-cricket-lightGold to-cricket-gold drop-shadow-lg mb-2 uppercase">Hand<br />Cricket</h1>
        </motion.div>

        {!hasName ? (
          <motion.div initial={{ scale: 0.9, opacity: 0 }} animate={{ scale: 1, opacity: 1 }} className="w-full bg-cricket-dark p-6 rounded-2xl border border-cricket-red/30">
            <h2 className="text-xl font-bold text-white mb-4 uppercase text-center tracking-widest">Enter Player Name</h2>
            <form onSubmit={saveName} className="flex flex-col gap-4">
              <input type="text" value={name} onChange={(e) => setName(e.target.value)} placeholder="E.g. Captain Cool" className="w-full bg-cricket-black border border-gray-700 rounded-xl p-4 text-white focus:outline-none focus:border-cricket-gold" maxLength={15} required />
              <button type="submit" className="w-full bg-cricket-gold text-black font-bold uppercase p-4 rounded-xl hover:bg-cricket-lightGold transition-colors">Let's Play</button>
            </form>
          </motion.div>
        ) : (
          <motion.div className="flex flex-col w-full gap-4" initial={{ y: 50, opacity: 0 }} animate={{ y: 0, opacity: 1 }} transition={{ duration: 0.8, delay: 0.2, ease: "easeOut" }}>
            <div className="text-center mb-2">
              <span className="text-gray-500 uppercase tracking-widest text-xs font-bold">Welcome back,</span>
              <p className="text-cricket-lightGold font-black uppercase tracking-wider">{name}</p>
            </div>
            
            <Link href="/play/ai" className="w-full">
              <motion.button whileHover={{ scale: 1.05 }} whileTap={{ scale: 0.95 }} className="w-full relative group overflow-hidden rounded-xl bg-gradient-to-r from-cricket-darkRed to-cricket-red p-4 shadow-[0_0_20px_rgba(211,47,47,0.4)] transition-all hover:shadow-[0_0_30px_rgba(211,47,47,0.6)]">
                <div className="absolute inset-0 bg-white/20 translate-y-full group-hover:translate-y-0 transition-transform duration-300 ease-out"></div>
                <div className="relative flex items-center justify-center gap-3">
                  <Play fill="currentColor" size={24} className="text-white" />
                  <span className="text-xl font-bold text-white uppercase tracking-wider">Play vs AI</span>
                </div>
              </motion.button>
            </Link>

            <Link href="/play/multiplayer" className="w-full">
              <motion.button whileHover={{ scale: 1.05 }} whileTap={{ scale: 0.95 }} className="w-full relative group overflow-hidden rounded-xl bg-cricket-dark border border-cricket-red/30 p-4 transition-all hover:border-cricket-red hover:shadow-[0_0_20px_rgba(211,47,47,0.2)]">
                <div className="absolute inset-0 bg-cricket-red/10 translate-y-full group-hover:translate-y-0 transition-transform duration-300 ease-out"></div>
                <div className="relative flex items-center justify-center gap-3">
                  <Users size={24} className="text-cricket-lightGold" />
                  <span className="text-xl font-bold text-white uppercase tracking-wider">Multiplayer</span>
                </div>
              </motion.button>
            </Link>

            <Link href="/stats" className="w-full">
              <motion.button whileHover={{ scale: 1.05 }} whileTap={{ scale: 0.95 }} className="w-full relative group overflow-hidden rounded-xl bg-cricket-dark border border-cricket-gold/30 p-4 transition-all hover:border-cricket-gold hover:shadow-[0_0_20px_rgba(255,215,0,0.2)]">
                <div className="absolute inset-0 bg-cricket-gold/10 translate-y-full group-hover:translate-y-0 transition-transform duration-300 ease-out"></div>
                <div className="relative flex items-center justify-center gap-3">
                  <Trophy size={24} className="text-cricket-gold" />
                  <span className="text-xl font-bold text-white uppercase tracking-wider">Leaderboard</span>
                </div>
              </motion.button>
            </Link>
          </motion.div>
        )}
      </div>
    </main>
  );
}
```

*(Note: AI Game, Multiplayer Game, and Leaderboard components are omitted for brevity as they are extremely long, but they are fully integrated in your workspace files!)*

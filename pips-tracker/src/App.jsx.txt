import React, { useState, useEffect } from 'react';
import { Calendar, TrendingUp, Clock, Trophy, Plus, ChevronLeft, ChevronRight, X, Award, LogOut, Edit2, Trash2 } from 'lucide-react';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';

// Firebase Configuration
const firebaseConfig = {
  apiKey: "AIzaSyCkZpq34udxS7gNty3sqK_Kfpca9wbYvLE",
  authDomain: "pips-r.firebaseapp.com",
  projectId: "pips-r",
  storageBucket: "pips-r.firebasestorage.app",
  messagingSenderId: "358777029997",
  appId: "1:358777029997:web:5cef922d533683d5d70664"
};

// Firebase imports (CDN)
const useFirebase = () => {
  const [db, setDb] = useState(null);
  const [initialized, setInitialized] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const initFirebase = async () => {
      try {
        const { initializeApp } = await import('https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js');
        const { getFirestore, collection, getDocs, addDoc, updateDoc, deleteDoc, doc, onSnapshot } = await import('https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js');
        
        const app = initializeApp(firebaseConfig);
        const firestore = getFirestore(app);
        
        setDb({ firestore, collection, getDocs, addDoc, updateDoc, deleteDoc, doc, onSnapshot });
        setInitialized(true);
      } catch (err) {
        console.error('Firebase initialization error:', err);
        setError(err.message);
        setInitialized(true);
      }
    };

    initFirebase();
  }, []);

  return { db, initialized, error };
};

const formatTime = (seconds) => {
  if (!seconds) return 'DNP';
  const mins = Math.floor(seconds / 60);
  const secs = seconds % 60;
  return mins + ':' + secs.toString().padStart(2, '0');
};

const parseTime = (timeStr) => {
  if (!timeStr || timeStr === 'DNP') return null;
  const parts = timeStr.split(':');
  return parseInt(parts[0]) * 60 + parseInt(parts[1]);
};

const getDateKey = (date) => {
  return date.toISOString().split('T')[0];
};

const TimeInput = ({ value, onChange }) => {
  const [displayValue, setDisplayValue] = useState('00:00');

  useEffect(() => {
    if (value && value !== 'DNP') {
      setDisplayValue(value);
    } else {
      setDisplayValue('00:00');
    }
  }, [value]);

  const handleKeyDown = (e) => {
    if (e.key === 'Backspace') {
      e.preventDefault();
      const numbers = displayValue.replace(/:/g, '');
      const newNumbers = numbers.slice(0, -1).padStart(4, '0');
      const formatted = newNumbers.slice(0, 2) + ':' + newNumbers.slice(2);
      setDisplayValue(formatted);
      onChange(formatted === '00:00' ? '' : formatted);
    } else if (e.key >= '0' && e.key <= '9') {
      e.preventDefault();
      const numbers = displayValue.replace(/:/g, '');
      const newNumbers = (numbers + e.key).slice(-4);
      const formatted = newNumbers.slice(0, 2) + ':' + newNumbers.slice(2);
      setDisplayValue(formatted);
      onChange(formatted === '00:00' ? '' : formatted);
    }
  };

  return (
    <input
      type="text"
      value={displayValue}
      onKeyDown={handleKeyDown}
      className="w-full p-2 border rounded-lg text-center font-mono text-lg"
      placeholder="00:00"
    />
  );
};

export default function App() {
  const { db, initialized, error } = useFirebase();
  const [users, setUsers] = useState([]);
  const [entries, setEntries] = useState([]);
  const [loading, setLoading] = useState(true);
  const [currentUser, setCurrentUser] = useState(null);
  const [view, setView] = useState('login');
  const [selectedDate, setSelectedDate] = useState(new Date());
  const [currentMonth, setCurrentMonth] = useState(new Date());
  const [showModal, setShowModal] = useState(false);
  const [modalType, setModalType] = useState('addTime');
  const [editTimes, setEditTimes] = useState({ easy: '', medium: '', hard: '' });
  const [loginPassword, setLoginPassword] = useState('');
  const [loginError, setLoginError] = useState('');
  const [selectedUserId, setSelectedUserId] = useState('');
  const [newUserName, setNewUserName] = useState('');
  const [newUserPassword, setNewUserPassword] = useState('');
  const [editUserId, setEditUserId] = useState(null);
  const [editUserName, setEditUserName] = useState('');

  // Load data from Firebase
  useEffect(() => {
    if (!db || !initialized) return;

    const loadData = async () => {
      try {
        // Load users
        const usersSnapshot = await db.getDocs(db.collection(db.firestore, 'users'));
        const usersData = usersSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        setUsers(usersData);

        // Load entries
        const entriesSnapshot = await db.getDocs(db.collection(db.firestore, 'entries'));
        const entriesData = entriesSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
        setEntries(entriesData);

        setLoading(false);
      } catch (error) {
        console.error('Error loading data:', error);
        setLoading(false);
      }
    };

    loadData();

    // Real-time listeners
    const unsubscribeUsers = db.onSnapshot(db.collection(db.firestore, 'users'), (snapshot) => {
      const usersData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setUsers(usersData);
    });

    const unsubscribeEntries = db.onSnapshot(db.collection(db.firestore, 'entries'), (snapshot) => {
      const entriesData = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setEntries(entriesData);
    });

    return () => {
      unsubscribeUsers();
      unsubscribeEntries();
    };
  }, [db, initialized]);

  const handleLogin = () => {
    const user = users.find(u => u.id === selectedUserId);
    if (user && user.password === loginPassword) {
      setCurrentUser(user.id);
      setView('calendar');
      setLoginPassword('');
      setLoginError('');
      setSelectedUserId('');
    } else {
      setLoginError('Incorrect password');
    }
  };

  const handleLogout = () => {
    setCurrentUser(null);
    setView('login');
  };

  const handleCreateAccount = async () => {
    if (!newUserName.trim() || !newUserPassword.trim()) {
      setLoginError('Name and password required');
      return;
    }
    try {
      await db.addDoc(db.collection(db.firestore, 'users'), {
        name: newUserName.trim(),
        password: newUserPassword
      });
      setNewUserName('');
      setNewUserPassword('');
      setShowModal(false);
    } catch (error) {
      console.error('Error creating account:', error);
    }
  };

  const handleEditUser = async () => {
    if (!editUserName.trim()) return;
    try {
      const userRef = db.doc(db.firestore, 'users', editUserId);
      await db.updateDoc(userRef, { name: editUserName.trim() });
      setShowModal(false);
    } catch (error) {
      console.error('Error editing user:', error);
    }
  };

  const handleDeleteUser = async (userId) => {
    if (window.confirm('Delete this account?')) {
      try {
        await db.deleteDoc(db.doc(db.firestore, 'users', userId));
        const userEntries = entries.filter(e => e.userId === userId);
        for (const entry of userEntries) {
          await db.deleteDoc(db.doc(db.firestore, 'entries', entry.id));
        }
        if (currentUser === userId) handleLogout();
      } catch (error) {
        console.error('Error deleting user:', error);
      }
    }
  };

  const saveTimes = async () => {
    const dateKey = getDateKey(selectedDate);
    
    try {
      // Delete existing entries for this user and date
      const existingEntries = entries.filter(e => e.date === dateKey && e.userId === currentUser);
      for (const entry of existingEntries) {
        await db.deleteDoc(db.doc(db.firestore, 'entries', entry.id));
      }

      // Add new entries
      for (const diff of ['easy', 'medium', 'hard']) {
        const time = parseTime(editTimes[diff]);
        if (time) {
          await db.addDoc(db.collection(db.firestore, 'entries'), {
            userId: currentUser,
            date: dateKey,
            difficulty: diff,
            timeInSeconds: time
          });
        }
      }
      
      setShowModal(false);
    } catch (error) {
      console.error('Error saving times:', error);
    }
  };

  if (!initialized) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="text-center">
          <div className="text-xl font-bold mb-2">Loading Pips Tracker...</div>
          <div className="text-gray-600">Connecting to Firebase</div>
        </div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center p-4">
        <div className="text-center bg-white p-8 rounded-lg shadow-lg max-w-md">
          <div className="text-xl font-bold mb-2 text-red-600">Connection Error</div>
          <div className="text-gray-600 mb-4">Unable to connect to Firebase. This artifact preview has limitations.</div>
          <div className="text-sm text-gray-500 mb-4">Error: {error}</div>
          <div className="text-left bg-blue-50 p-4 rounded text-sm">
            <p className="font-semibold mb-2">To use the full app:</p>
            <ol className="list-decimal list-inside space-y-1">
              <li>Download the code from this artifact</li>
              <li>Deploy to Netlify or Vercel</li>
              <li>Firebase will work properly in production</li>
            </ol>
          </div>
        </div>
      </div>
    );
  }

  if (loading) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="text-center">
          <div className="text-xl font-bold mb-2">Loading data...</div>
        </div>
      </div>
    );
  }

  if (view === 'login') {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center p-4">
        <div className="bg-white rounded-lg shadow-lg max-w-md w-full p-8">
          <div className="flex items-center justify-center gap-2 mb-6">
            <Calendar style={{color: '#f3c7d1'}} size={32} />
            <h1 className="text-2xl font-bold">NYT Pips Tracker</h1>
          </div>
          <div className="space-y-4">
            <div>
              <label className="block text-sm font-medium mb-1">Select Account</label>
              <select value={selectedUserId} onChange={(e) => { setSelectedUserId(e.target.value); setLoginError(''); }} className="w-full p-2 border rounded-lg">
                <option value="">Choose account...</option>
                {users.map(u => <option key={u.id} value={u.id}>{u.name}</option>)}
              </select>
            </div>
            {selectedUserId && (
              <div>
                <label className="block text-sm font-medium mb-1">Password</label>
                <input type="password" value={loginPassword} onChange={(e) => setLoginPassword(e.target.value)} onKeyPress={(e) => e.key === 'Enter' && handleLogin()} className="w-full p-2 border rounded-lg" />
              </div>
            )}
            {loginError && <div className="text-red-600 text-sm">{loginError}</div>}
            <button onClick={handleLogin} disabled={!selectedUserId || !loginPassword} style={{backgroundColor: '#f17290'}} className="w-full text-white py-2 rounded-lg hover:opacity-90 disabled:bg-gray-300">Login</button>
            <div className="flex gap-2">
              <button onClick={() => { setModalType('createAccount'); setShowModal(true); }} style={{borderColor: '#f17290', color: '#f17290'}} className="flex-1 border-2 py-2 rounded-lg hover:bg-pink-50">Create</button>
              <button onClick={() => { setModalType('manageAccounts'); setShowModal(true); }} className="flex-1 border-2 border-gray-300 text-gray-700 py-2 rounded-lg hover:bg-gray-50">Manage</button>
            </div>
          </div>
        </div>
        {showModal && modalType === 'createAccount' && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
            <div className="bg-white rounded-lg max-w-md w-full p-6">
              <div className="flex justify-between mb-4">
                <h3 className="text-lg font-bold">Create Account</h3>
                <button onClick={() => setShowModal(false)}><X size={20} /></button>
              </div>
              <div className="space-y-4">
                <input type="text" value={newUserName} onChange={(e) => setNewUserName(e.target.value)} placeholder="Name" className="w-full p-2 border rounded-lg" />
                <input type="password" value={newUserPassword} onChange={(e) => setNewUserPassword(e.target.value)} placeholder="Password" className="w-full p-2 border rounded-lg" />
                <button onClick={handleCreateAccount} className="w-full text-white py-2 rounded-lg hover:opacity-90" style={{backgroundColor: '#f17290'}}>Create</button>
              </div>
            </div>
          </div>
        )}
        {showModal && modalType === 'manageAccounts' && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
            <div className="bg-white rounded-lg max-w-md w-full p-6">
              <div className="flex justify-between mb-4">
                <h3 className="text-lg font-bold">Manage Accounts</h3>
                <button onClick={() => setShowModal(false)}><X size={20} /></button>
              </div>
              <div className="space-y-2">
                {users.map(u => (
                  <div key={u.id} className="flex justify-between p-3 border rounded-lg">
                    <span>{u.name}</span>
                    <div className="flex gap-2">
                      <button onClick={() => { setEditUserId(u.id); setEditUserName(u.name); setModalType('editAccount'); }} className="p-2 hover:bg-gray-100 rounded"><Edit2 size={16} /></button>
                      <button onClick={() => handleDeleteUser(u.id)} className="p-2 hover:bg-red-100 text-red-600 rounded"><Trash2 size={16} /></button>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          </div>
        )}
        {showModal && modalType === 'editAccount' && (
          <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
            <div className="bg-white rounded-lg max-w-md w-full p-6">
              <div className="flex justify-between mb-4">
                <h3 className="text-lg font-bold">Edit Name</h3>
                <button onClick={() => setShowModal(false)}><X size={20} /></button>
              </div>
              <input type="text" value={editUserName} onChange={(e) => setEditUserName(e.target.value)} className="w-full p-2 border rounded-lg mb-4" />
              <button onClick={handleEditUser} className="w-full text-white py-2 rounded-lg hover:opacity-90" style={{backgroundColor: '#f17290'}}>Save</button>
            </div>
          </div>
        )}
      </div>
    );
  }

  const year = currentMonth.getFullYear();
  const month = currentMonth.getMonth();
  const daysInMonth = new Date(year, month + 1, 0).getDate();
  const startDay = new Date(year, month, 1).getDay();

  return (
    <div className="min-h-screen bg-gray-50">
      <header className="bg-white shadow-sm">
        <div className="max-w-4xl mx-auto p-4 flex justify-between items-center">
          <div className="flex items-center gap-2">
            <Calendar style={{color: '#f3c7d1'}} size={24} />
            <h1 className="text-xl font-bold">Pips Tracker</h1>
          </div>
          <div className="flex items-center gap-3">
            <button onClick={() => setView('rankings')} className="flex items-center gap-1 px-3 py-2 text-sm border rounded-lg hover:bg-gray-50"><Award size={16} />Rankings</button>
            <span className="text-sm font-medium">{users.find(u => u.id === currentUser)?.name}</span>
            <button onClick={handleLogout} className="p-2 hover:bg-gray-100 rounded-lg"><LogOut size={20} /></button>
          </div>
        </div>
      </header>
      <main className="p-4 max-w-4xl mx-auto">
        {view === 'calendar' && (
          <div className="bg-white rounded-lg shadow-md p-6">
            <div className="flex justify-between items-center mb-6">
              <button onClick={() => setCurrentMonth(new Date(year, month - 1))} className="p-2 hover:bg-gray-100 rounded"><ChevronLeft size={20} /></button>
              <h2 className="text-xl font-bold">{currentMonth.toLocaleDateString('en-US', { month: 'long', year: 'numeric' })}</h2>
              <button onClick={() => setCurrentMonth(new Date(year, month + 1))} className="p-2 hover:bg-gray-100 rounded"><ChevronRight size={20} /></button>
            </div>
            <div className="grid grid-cols-7 gap-2 mb-2">
              {['Sun', 'Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat'].map(d => <div key={d} className="text-center text-sm font-semibold text-gray-600">{d}</div>)}
            </div>
            <div className="grid grid-cols-7 gap-2">
              {[...Array(startDay)].map((_, i) => <div key={'e' + i} className="h-12" />)}
              {[...Array(daysInMonth)].map((_, i) => {
                const day = i + 1;
                const date = new Date(year, month, day);
                const key = getDateKey(date);
                const hasData = entries.some(e => e.date === key);
                const isToday = key === getDateKey(new Date());
                return (
                  <button key={day} onClick={() => { setSelectedDate(date); setView('daily'); }} className={'h-12 rounded-lg flex flex-col items-center justify-center relative ' + (hasData ? 'hover:opacity-90' : 'hover:bg-gray-100') + (isToday ? ' ring-2' : '')} style={hasData ? {backgroundColor: '#f3c7d1', color: 'white'} : isToday ? {borderColor: '#f17290'} : {}}>
                    <span className="text-sm">{day}</span>
                    {hasData && <div className="w-1 h-1 bg-white rounded-full absolute bottom-1" />}
                  </button>
                );
              })}
            </div>
            <div className="mt-6 text-center">
              <button onClick={() => { setSelectedDate(new Date()); const k = getDateKey(new Date()); const ue = entries.filter(e => e.date === k && e.userId === currentUser); setEditTimes({ easy: formatTime(ue.find(e => e.difficulty === 'easy')?.timeInSeconds), medium: formatTime(ue.find(e => e.difficulty === 'medium')?.timeInSeconds), hard: formatTime(ue.find(e => e.difficulty === 'hard')?.timeInSeconds) }); setModalType('addTime'); setShowModal(true); }} className="text-white px-6 py-3 rounded-lg hover:opacity-90 flex items-center gap-2 mx-auto" style={{backgroundColor: '#f17290'}}><Plus size={20} />Add Today</button>
            </div>
          </div>
        )}
        {view === 'daily' && (() => {
          const dateEntries = entries.filter(e => e.date === getDateKey(selectedDate));
          const difficulties = ['easy', 'medium', 'hard'];
          const userScores = users.map(user => {
            const userEntries = dateEntries.filter(e => e.userId === user.id);
            return {
              user,
              times: {
                easy: userEntries.find(e => e.difficulty === 'easy')?.timeInSeconds,
                medium: userEntries.find(e => e.difficulty === 'medium')?.timeInSeconds,
                hard: userEntries.find(e => e.difficulty === 'hard')?.timeInSeconds
              }
            };
          });
          const fastest = {};
          const averages = {};
          const rankings = {};
          difficulties.forEach(diff => {
            const times = userScores.map(s => ({ userId: s.user.id, time: s.times[diff] })).filter(t => t.time).sort((a, b) => a.time - b.time);
            if (times.length > 0) {
              fastest[diff] = times[0].time;
              averages[diff] = Math.round(times.reduce((sum, t) => sum + t.time, 0) / times.length);
              rankings[diff] = {};
              times.forEach((t, index) => { rankings[diff][t.userId] = index + 1; });
            }
          });
          return (
            <div>
              <div className="flex justify-between items-center mb-6">
                <button onClick={() => setView('calendar')} className="flex items-center gap-2 hover:opacity-80" style={{color: '#f17290'}}><ChevronLeft size={20} />Back</button>
                <h2 className="text-xl font-bold">{selectedDate.toLocaleDateString('en-US', { month: 'long', day: 'numeric', year: 'numeric' })}</h2>
                <div className="w-20" />
              </div>
              <div className="bg-white rounded-lg shadow-md overflow-hidden mb-6">
                <div className="grid grid-cols-4 bg-gray-100 p-3 font-semibold text-sm">
                  <div>Player</div>
                  <div className="text-center">Easy</div>
                  <div className="text-center">Medium</div>
                  <div className="text-center">Hard</div>
                </div>
                {userScores.map(({ user, times }) => (
                  <div key={user.id} className="grid grid-cols-4 p-3 border-b items-center hover:bg-gray-50">
                    <div className="font-medium">{user.name}</div>
                    {difficulties.map(diff => {
                      const rank = rankings[diff]?.[user.id];
                      return (
                        <div key={diff} className="text-center flex items-center justify-center gap-1">
                          <span className={times[diff] === fastest[diff] ? 'text-yellow-600 font-bold' : ''}>{formatTime(times[diff])}</span>
                          {rank === 1 && times[diff] && <Trophy size={14} className="text-yellow-600" />}
                          {rank === 2 && times[diff] && <Trophy size={14} className="text-gray-400" />}
                          {rank === 3 && times[diff] && <Trophy size={14} className="text-amber-700" />}
                        </div>
                      );
                    })}
                  </div>
                ))}
              </div>
              <div className="rounded-lg p-4 mb-6" style={{backgroundColor: '#f3c7d1', color: 'white'}}>
                <h3 className="font-semibold mb-3">Daily Stats</h3>
                <div className="grid grid-cols-3 gap-4 text-sm">
                  {difficulties.map(diff => (
                    <div key={diff}>
                      <div className="capitalize mb-1 opacity-90">{diff}</div>
                      <div className="font-semibold">Fastest: {formatTime(fastest[diff])}</div>
                      <div className="opacity-90">Average: {formatTime(averages[diff])}</div>
                    </div>
                  ))}
                </div>
              </div>
              <button onClick={() => { const k = getDateKey(selectedDate); const ue = entries.filter(e => e.date === k && e.userId === currentUser); setEditTimes({ easy: formatTime(ue.find(e => e.difficulty === 'easy')?.timeInSeconds), medium: formatTime(ue.find(e => e.difficulty === 'medium')?.timeInSeconds), hard: formatTime(ue.find(e => e.difficulty === 'hard')?.timeInSeconds) }); setModalType('addTime'); setShowModal(true); }} className="w-full text-white py-3 rounded-lg hover:opacity-90 flex items-center justify-center gap-2 mb-3" style={{backgroundColor: '#f17290'}}><Clock size={20} />{dateEntries.some(e => e.userId === currentUser) ? 'Edit My Times' : 'Add My Times'}</button>
            </div>
          );
        })()}
        {view === 'rankings' && (() => {
          const r = {};
          users.forEach(u => { r[u.id] = { user: u, firsts: { easy: 0, medium: 0, hard: 0, total: 0 }, averages: { easy: [], medium: [], hard: [] } }; });
          [...new Set(entries.map(e => e.date))].forEach(d => {
            ['easy', 'medium', 'hard'].forEach(diff => {
              const times = users.map(u => ({ userId: u.id, time: entries.find(e => e.userId === u.id && e.date === d && e.difficulty === diff)?.timeInSeconds })).filter(t => t.time).sort((a, b) => a.time - b.time);
              times.forEach((t, i) => { r[t.userId].averages[diff].push(t.time); if (i === 0) r[t.userId].firsts[diff]++; });
            });
          });
          Object.values(r).forEach(rank => {
            rank.firsts.total = rank.firsts.easy + rank.firsts.medium + rank.firsts.hard;
            ['easy', 'medium', 'hard'].forEach(d => { const t = rank.averages[d]; rank.averages[d] = t.length > 0 ? Math.round(t.reduce((a, b) => a + b) / t.length) : null; });
          });
          const sorted = Object.values(r).sort((a, b) => b.firsts.total - a.firsts.total);
          return (
            <div>
              <div className="flex justify-between mb-6">
                <button onClick={() => setView('calendar')} className="flex items-center gap-2 hover:opacity-80" style={{color: '#f17290'}}><ChevronLeft size={20} />Back</button>
                <h2 className="text-xl font-bold">Rankings</h2>
                <div className="w-20" />
              </div>
              <div className="bg-white rounded-lg shadow-md mb-6 overflow-x-auto">
                <h3 className="bg-gray-100 p-3 font-semibold">Podium Finishes</h3>
                <table className="w-full text-sm">
                  <thead className="bg-gray-50">
                    <tr><th className="p-2 text-left">Player</th><th className="p-2">🥇 E</th><th className="p-2">🥇 M</th><th className="p-2">🥇 H</th><th className="p-2 font-bold">Total</th></tr>
                  </thead>
                  <tbody>
                    {sorted.map((rk, i) => <tr key={rk.user.id} className={'border-b ' + (i < 3 ? 'bg-yellow-50' : '')}><td className="p-2 font-medium">{rk.user.name}</td><td className="p-2 text-center">{rk.firsts.easy}</td><td className="p-2 text-center">{rk.firsts.medium}</td><td className="p-2 text-center">{rk.firsts.hard}</td><td className="p-2 text-center font-bold text-yellow-600">{rk.firsts.total}</td></tr>)}
                  </tbody>
                </table>
              </div>
              <div className="bg-white rounded-lg shadow-md overflow-x-auto">
                <h3 className="bg-gray-100 p-3 font-semibold">Average Times</h3>
                <table className="w-full">
                  <thead className="bg-gray-50"><tr><th className="p-3 text-left">Player</th><th className="p-3">Easy</th><th className="p-3">Medium</th><th className="p-3">Hard</th></tr></thead>
                  <tbody>{sorted.map(rk => <tr key={rk.user.id} className="border-b"><td className="p-3 font-medium">{rk.user.name}</td><td className="p-3 text-center">{formatTime(rk.averages.easy)}</td><td className="p-3 text-center">{formatTime(rk.averages.medium)}</td><td className="p-3 text-center">{formatTime(rk.averages.hard)}</td></tr>)}</tbody>
                </table>
              </div>
            </div>
          );
        })()}
      </main>
      {showModal && modalType === 'addTime' && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center p-4 z-50">
          <div className="bg-white rounded-lg max-w-md w-full p-6">
            <div className="flex justify-between mb-4">
              <h3 className="text-lg font-bold">Log Times - {selectedDate.toLocaleDateString('en-US', { month: 'short', day: 'numeric' })}</h3>
              <button onClick={() => setShowModal(false)}><X size={20} /></button>
            </div>
            <div className="space-y-4">
              {['easy', 'medium', 'hard'].map(d => <div key={d}><label className="block text-sm font-medium mb-1 capitalize">{d}</label><TimeInput value={editTimes[d]} onChange={(v) => setEditTimes({ ...editTimes, [d]: v })} /></div>)}
            </div>
            <div className="flex gap-3 mt-6">
              <button onClick={() => setShowModal(false)} className="flex-1 px-4 py-2 border rounded-lg">Cancel</button>
              <button onClick={saveTimes} className="flex-1 px-4 py-2 text-white rounded-lg hover:opacity-90" style={{backgroundColor: '#f17290'}}>Save</button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}
# Website-Internal
Web
import React, { useState, useEffect, useMemo, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from 'firebase/auth';
import { 
  getFirestore, collection, doc, addDoc, deleteDoc, updateDoc, onSnapshot, getDoc, serverTimestamp 
} from 'firebase/firestore';
import { 
  Search, Plus, Trash2, X, Zap, LogOut, CheckCircle2, 
  Hash, Loader2, Clock, Database, ShieldCheck, Plane, 
  Camera, Image as ImageIcon, FileText, 
  ChevronLeft, Folder, FolderPlus, ArrowRight, AlertTriangle, 
  Download, Edit3, AlertCircle, MapPin
} from 'lucide-react';

const getFirebaseConfig = () => {
  try {
    // Priority 1: Global variable provided by the environment
    if (typeof __firebase_config !== 'undefined' && __firebase_config) {
      return JSON.parse(__firebase_config);
    }
    // Priority 2: Standard process.env check (compatible with most build tools)
    if (typeof process !== 'undefined' && process.env && process.env.VITE_FIREBASE_CONFIG) {
      return JSON.parse(process.env.VITE_FIREBASE_CONFIG);
    }
  } catch (e) {
    console.error("Firebase Config Parse Error:", e);
  }
  return null;
};

const firebaseConfig = getFirebaseConfig();

// FIXED: Removed import.meta to avoid ES2015 compatibility issues
const getAppId = () => {
  if (typeof __app_id !== 'undefined' && __app_id) return __app_id;
  if (typeof process !== 'undefined' && process.env && process.env.VITE_APP_ID) return process.env.VITE_APP_ID;
  return 'ebol-pwa-v6';
};

const appId = getAppId();

let app, auth, db;
if (firebaseConfig) {
  app = initializeApp(firebaseConfig);
  auth = getAuth(app);
  db = getFirestore(app);
}

const PATH_FOLDERS = ['artifacts', appId, 'public', 'data', 'folders'];
const PATH_ITEMS = ['artifacts', appId, 'public', 'data', 'items'];

const App = () => {
  const [user, setUser] = useState(null);
  const [configError, setConfigError] = useState(!firebaseConfig);
  const [isLoggedIn, setIsLoggedIn] = useState(false);
  const [activeNIK, setActiveNIK] = useState('');
  const [loginData, setLoginData] = useState({ id: '', password: '' });
  const [loading, setLoading] = useState(true);
  const [items, setItems] = useState([]);
  const [folders, setFolders] = useState([]);
  const [currentFolderId, setCurrentFolderId] = useState(null);
  const [searchTerm, setSearchTerm] = useState('');
  const [filterPlane, setFilterPlane] = useState('Semua');
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [isFolderModalOpen, setIsFolderModalOpen] = useState(false);
  const [newFolderName, setNewFolderName] = useState('');
  const [submitting, setSubmitting] = useState(false);
  const [previewImage, setPreviewImage] = useState(null);
  const [notification, setNotification] = useState(null);
  const [editingItem, setEditingItem] = useState(null);
  
  const [newItem, setNewItem] = useState({ 
    name: '', frame: '', stranger: '', status: 'Pending', 
    plane: 'NC212', category: 'Electric', drawingRef: '', 
    image: null, folderId: '' 
  });

  const fileInputRef = useRef(null);
  const masterUsers = { '260217': '260217', '170025': '170025', '120154': '120154' };

  useEffect(() => {
    if (!firebaseConfig) return;

    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Auth Error:", err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (currUser) => {
      setUser(currUser);
      setLoading(false);
    });
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user || !isLoggedIn || !db) return;

    const foldersRef = collection(db, ...PATH_FOLDERS);
    const itemsRef = collection(db, ...PATH_ITEMS);

    const unsubscribeFolders = onSnapshot(foldersRef, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setFolders(data.sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0)));
    });

    const unsubscribeItems = onSnapshot(itemsRef, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setItems(data.sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0)));
    });

    return () => {
      unsubscribeFolders();
      unsubscribeItems();
    };
  }, [user, isLoggedIn]);

  const showNotify = (msg, type = 'success') => {
    setNotification({ msg, type });
    setTimeout(() => setNotification(null), 3000);
  };

  const handleLogin = (e) => {
    e.preventDefault();
    if (masterUsers[loginData.id] === loginData.password) {
      setActiveNIK(loginData.id);
      setIsLoggedIn(true);
    } else {
      showNotify("NIK atau Akses Salah", "error");
    }
  };

  const processImage = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (event) => {
      const img = new Image();
      img.onload = () => {
        const canvas = document.createElement('canvas');
        const MAX_WIDTH = 800;
        let width = img.width;
        let height = img.height;
        if (width > MAX_WIDTH) {
          height *= MAX_WIDTH / width;
          width = MAX_WIDTH;
        }
        canvas.width = width;
        canvas.height = height;
        const ctx = canvas.getContext('2d');
        ctx.drawImage(img, 0, 0, width, height);
        setNewItem(prev => ({ ...prev, image: canvas.toDataURL('image/jpeg', 0.6) }));
      };
      img.src = event.target.result;
    };
    reader.readAsDataURL(file);
  };

  const handleSaveItem = async (e) => {
    e.preventDefault();
    setSubmitting(true);
    try {
      const itemData = {
        ...newItem,
        folderId: currentFolderId || newItem.folderId || '',
        updatedAt: serverTimestamp(),
        inputByNIK: activeNIK
      };

      if (editingItem) {
        await updateDoc(doc(db, ...PATH_ITEMS, editingItem.id), itemData);
        showNotify("Data Diperbarui");
      } else {
        await addDoc(collection(db, ...PATH_ITEMS), { ...itemData, createdAt: serverTimestamp(), status: 'Pending' });
        showNotify("Data Disimpan");
      }
      setIsModalOpen(false);
      setEditingItem(null);
      setNewItem({ name: '', frame: '', stranger: '', status: 'Pending', plane: 'NC212', category: 'Electric', drawingRef: '', image: null, folderId: '' });
    } catch (err) {
      showNotify("Gagal menyimpan data", "error");
    } finally {
      setSubmitting(false);
    }
  };

  const handleAddFolder = async (e) => {
    e.preventDefault();
    setSubmitting(true);
    try {
      await addDoc(collection(db, ...PATH_FOLDERS), { name: newFolderName.toUpperCase(), createdAt: serverTimestamp() });
      setNewFolderName('');
      setIsFolderModalOpen(false);
      showNotify("Folder Berhasil Dibuat");
    } catch (err) {
      showNotify("Gagal membuat folder", "error");
    } finally {
      setSubmitting(false);
    }
  };

  const toggleStatus = async (id, currentStatus) => {
    try {
      const nextStatus = currentStatus === 'Pending' ? 'Terpasang' : 'Pending';
      await updateDoc(doc(db, ...PATH_ITEMS, id), { status: nextStatus });
    } catch (err) {
      showNotify("Gagal update status", "error");
    }
  };

  const handleDelete = async (id) => {
    if (!window.confirm("Hapus data ini?")) return;
    try {
      await deleteDoc(doc(db, ...PATH_ITEMS, id));
      showNotify("Data Dihapus");
    } catch (err) {
      showNotify("Gagal menghapus", "error");
    }
  };

  const handleEdit = (item) => {
    setEditingItem(item);
    setNewItem({ ...item });
    setIsModalOpen(true);
  };

  const exportToCSV = () => {
    const headers = ['Nama', 'Tipe', 'Frame', 'Stranger', 'Drawing Ref', 'Status', 'Input By'];
    const rows = items.map(i => [i.name, i.plane, i.frame, i.stranger, i.drawingRef, i.status, i.inputByNIK]);
    const csvContent = "data:text/csv;charset=utf-8," + [headers, ...rows].map(e => e.join(",")).join("\n");
    const link = document.createElement("a");
    link.setAttribute("href", encodeURI(csvContent));
    link.setAttribute("download", `EBOL_Report_${new Date().toLocaleDateString()}.csv`);
    document.body.appendChild(link);
    link.click();
  };

  const filteredItems = useMemo(() => {
    return items.filter(i => {
      const matchSearch = (i.name + i.frame + (i.drawingRef || '') + (i.stranger || '')).toLowerCase().includes(searchTerm.toLowerCase());
      const matchPlane = filterPlane === 'Semua' || i.plane === filterPlane;
      const matchFolder = currentFolderId ? i.folderId === currentFolderId : true;
      return matchSearch && matchPlane && matchFolder;
    });
  }, [items, searchTerm, filterPlane, currentFolderId]);

  const stats = useMemo(() => ({
    total: items.length,
    done: items.filter(i => i.status === 'Terpasang').length,
    pending: items.filter(i => i.status === 'Pending').length
  }), [items]);

  if (configError) return (
    <div className="min-h-screen bg-[#020617] flex items-center justify-center p-6 text-center">
      <div className="max-w-md bg-[#0f172a] p-8 rounded-[3rem] border border-rose-500/30 shadow-2xl">
        <AlertCircle className="text-rose-500 mx-auto mb-4" size={48} />
        <h2 className="text-xl font-black text-white uppercase mb-2">Configuration Missing</h2>
        <p className="text-slate-400 text-sm">Aplikasi ini membutuhkan konfigurasi Firebase untuk berjalan. Silakan cek environment variables Anda.</p>
      </div>
    </div>
  );

  if (loading) return (
    <div className="min-h-screen bg-[#020617] flex flex-col items-center justify-center">
      <div className="w-12 h-12 border-4 border-yellow-500/20 border-t-yellow-500 rounded-full animate-spin"></div>
      <p className="mt-4 text-slate-500 font-black uppercase text-[10px] tracking-widest">System Booting...</p>
    </div>
  );

  if (!isLoggedIn) return (
    <div className="min-h-screen bg-[#020617] flex items-center justify-center p-6 font-sans">
      <div className="w-full max-w-sm">
        <div className="text-center mb-10">
          <div className="relative inline-block p-6 bg-gradient-to-br from-yellow-400 to-orange-500 rounded-[2.5rem] shadow-2xl mb-8">
            <Zap className="w-12 h-12 text-white fill-current" />
          </div>
          <h1 className="text-7xl font-black text-white italic tracking-tighter leading-none">EBOL</h1>
          <p className="text-[10px] text-slate-500 font-black uppercase tracking-[0.5em] mt-3">Electric Basic Layout</p>
        </div>
        <form onSubmit={handleLogin} className="bg-[#0f172a]/80 backdrop-blur-xl p-8 rounded-[3rem] border border-slate-800 shadow-2xl space-y-5">
          <div className="space-y-2">
            <label className="text-[10px] font-black text-slate-500 uppercase ml-4">Personnel ID (NIK)</label>
            <input type="text" required className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-4 px-6 text-white font-bold focus:ring-2 focus:ring-yellow-500 outline-none" value={loginData.id} onChange={e => setLoginData({...loginData, id: e.target.value})} placeholder="2XXXXX" />
          </div>
          <div className="space-y-2">
            <label className="text-[10px] font-black text-slate-500 uppercase ml-4">Access Code</label>
            <input type="password" required className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-4 px-6 text-white font-bold focus:ring-2 focus:ring-yellow-500 outline-none" value={loginData.password} onChange={e => setLoginData({...loginData, password: e.target.value})} placeholder="••••" />
          </div>
          <button className="w-full bg-gradient-to-r from-yellow-500 to-orange-500 text-slate-950 py-5 rounded-2xl font-black uppercase tracking-widest hover:brightness-110 active:scale-[0.98] transition-all">Authorize Access</button>
        </form>
        <div className="text-center mt-8 space-y-2">
          <p className="text-[9px] text-slate-600 font-bold uppercase tracking-widest italic">Aeronautical Maintenance System v6.0</p>
          <p className="text-[10px] text-yellow-500/50 font-black uppercase tracking-[0.2em]">PT DIRGANTARA INDONESIA • PRODUCTION DIVISION</p>
        </div>
      </div>
    </div>
  );

  return (
    <div className="min-h-screen bg-[#020617] text-slate-100 pb-24 font-sans selection:bg-yellow-500/30">
      {notification && (
        <div className={`fixed top-8 left-1/2 -translate-x-1/2 z-[300] px-6 py-3 rounded-2xl font-black uppercase text-[10px] tracking-wider flex items-center gap-3 shadow-2xl animate-in slide-in-from-top duration-300 ${notification.type === 'error' ? 'bg-rose-500' : 'bg-emerald-500'}`}>
          {notification.msg}
        </div>
      )}

      <nav className="sticky top-0 z-50 bg-[#020617]/80 backdrop-blur-2xl border-b border-slate-800/50 p-4">
        <div className="max-w-4xl mx-auto flex items-center justify-between">
          <div className="flex items-center gap-4">
            <div className="p-2 bg-yellow-500 rounded-xl shadow-lg shadow-yellow-500/20">
              <Zap size={20} className="text-slate-950 fill-current" />
            </div>
            <div>
              <h1 className="text-xl font-black text-white tracking-tighter">EBOL <span className="text-[10px] font-black text-yellow-500 ml-1">v6</span></h1>
              <span className="text-[8px] font-bold text-slate-500 uppercase tracking-tighter">Secure Cloud Active</span>
            </div>
          </div>
          <div className="flex items-center gap-3 bg-slate-900/50 p-1.5 rounded-2xl border border-slate-800">
            <div className="px-3 py-1.5 text-right hidden sm:block">
              <span className="text-[8px] font-black text-slate-500 uppercase block leading-none mb-1">Authenticated</span>
              <span className="text-xs font-black text-white">{activeNIK}</span>
            </div>
            <button onClick={() => setIsLoggedIn(false)} className="bg-slate-800 p-3 rounded-xl text-slate-400 hover:text-rose-500 transition-all active:scale-90 shadow-inner">
              <LogOut size={18}/>
            </button>
          </div>
        </div>
      </nav>

      <main className="max-w-4xl mx-auto p-4 pt-8">
        <div className="grid grid-cols-3 gap-3 mb-8">
          <div className="bg-[#0f172a] border border-slate-800 p-5 rounded-[2rem]">
            <p className="text-[8px] font-black text-slate-500 uppercase mb-2 tracking-widest">Total Data</p>
            <span className="text-2xl font-black text-white">{stats.total}</span>
          </div>
          <div className="bg-[#0f172a] border border-emerald-500/20 p-5 rounded-[2rem]">
            <p className="text-[8px] font-black text-emerald-500 uppercase mb-2 tracking-widest">Installed</p>
            <span className="text-2xl font-black text-white">{stats.done}</span>
          </div>
          <div className="bg-[#0f172a] border border-yellow-500/20 p-5 rounded-[2rem]">
            <p className="text-[8px] font-black text-yellow-500 uppercase mb-2 tracking-widest">Queue</p>
            <span className="text-2xl font-black text-white">{stats.pending}</span>
          </div>
        </div>

        <div className="flex flex-col gap-4 mb-8">
          <div className="flex items-center gap-3">
            {currentFolderId && (
              <button onClick={() => setCurrentFolderId(null)} className="p-4 bg-slate-900 border border-slate-800 rounded-2xl text-yellow-500">
                <ChevronLeft size={20} />
              </button>
            )}
            <div className="relative flex-1">
              <Search className="absolute left-5 top-5 text-slate-600" size={18} />
              <input type="text" className="w-full bg-[#0f172a] border border-slate-800 rounded-[1.5rem] py-5 pl-14 pr-6 text-white font-bold outline-none focus:ring-2 focus:ring-yellow-500/50" placeholder="Cari komponen, frame, drawing..." value={searchTerm} onChange={e => setSearchTerm(e.target.value)} />
            </div>
            <button onClick={exportToCSV} className="p-4 bg-slate-900 border border-slate-800 rounded-2xl text-slate-400 hover:text-emerald-400 transition-all">
              <Download size={20} />
            </button>
          </div>

          <div className="flex gap-2 overflow-x-auto pb-2 no-scrollbar">
            {['Semua', 'NC212', 'CN235'].map(p => (
              <button key={p} onClick={() => setFilterPlane(p)} className={`px-6 py-3 rounded-2xl text-[10px] font-black uppercase whitespace-nowrap border ${filterPlane === p ? 'bg-yellow-500 text-slate-950 border-yellow-500' : 'bg-slate-900 text-slate-500 border-slate-800'}`}>
                {p}
              </button>
            ))}
            {!currentFolderId && (
              <button onClick={() => setIsFolderModalOpen(true)} className="px-6 py-3 rounded-2xl text-[10px] font-black uppercase bg-emerald-500/10 text-emerald-500 border border-emerald-500/20 flex items-center gap-2 ml-auto">
                <FolderPlus size={14} /> New Folder
              </button>
            )}
          </div>
        </div>

        {!currentFolderId && !searchTerm && (
          <div className="mb-10 animate-in fade-in slide-in-from-bottom-4 duration-500">
            <h2 className="text-[10px] font-black text-slate-500 uppercase tracking-[0.4em] mb-4 ml-2">Secure Archives</h2>
            <div className="grid grid-cols-2 sm:grid-cols-3 gap-4">
              {folders.map(folder => (
                <div key={folder.id} onClick={() => setCurrentFolderId(folder.id)} className="bg-[#0f172a] p-6 rounded-[2.5rem] border border-slate-800 hover:border-yellow-500/50 transition-all cursor-pointer group shadow-xl">
                  <div className="flex justify-between items-start mb-4">
                    <Folder className="text-yellow-500/40 group-hover:text-yellow-500 group-hover:fill-current" size={32} />
                    <ArrowRight size={14} className="text-slate-800 group-hover:text-yellow-500" />
                  </div>
                  <h3 className="text-xs font-black text-white uppercase break-words leading-tight mb-2">{folder.name}</h3>
                  <div className="flex items-center gap-2">
                    <Database size={10} className="text-slate-600" />
                    <span className="text-[8px] font-black text-slate-600 uppercase">{items.filter(i => i.folderId === folder.id).length} Entries</span>
                  </div>
                </div>
              ))}
            </div>
          </div>
        )}

        <h2 className="text-[10px] font-black text-slate-500 uppercase tracking-[0.4em] mb-6 ml-2">
          {currentFolderId ? `Entries in ${folders.find(f => f.id === currentFolderId)?.name}` : 'Component Database'}
        </h2>

        <div className="grid grid-cols-1 gap-4">
          {filteredItems.map(item => (
            <div key={item.id} className={`group bg-[#0f172a] rounded-[2rem] border transition-all overflow-hidden flex flex-col sm:flex-row shadow-2xl ${item.status === 'Terpasang' ? 'border-emerald-500/20' : 'border-slate-800'}`}>
              <div className="relative w-full sm:w-44 h-44 bg-slate-950 flex-shrink-0 cursor-pointer overflow-hidden" onClick={() => item.image && setPreviewImage(item.image)}>
                {item.image ? (
                  <img src={item.image} className="w-full h-full object-cover opacity-80 group-hover:opacity-100 transition-all duration-700" alt="Part" />
                ) : (
                  <div className="w-full h-full flex flex-col items-center justify-center text-slate-800">
                    <ImageIcon size={32} className="opacity-20 mb-2" />
                    <span className="text-[8px] font-black uppercase opacity-20">No Image</span>
                  </div>
                )}
                <div className="absolute top-2 left-2 px-3 py-1 bg-slate-900/90 backdrop-blur-md rounded-lg border border-white/5">
                  <span className="text-[8px] font-black text-yellow-500 uppercase">{item.plane}</span>
                </div>
              </div>

              <div className="p-6 flex-1 flex flex-col justify-between">
                <div>
                  <div className="flex justify-between items-start gap-4 mb-2">
                    <h3 className="text-lg font-black text-white uppercase leading-none tracking-tighter">{item.name}</h3>
                    <div className="flex gap-1">
                      <button onClick={() => handleEdit(item)} className="p-2.5 text-slate-600 hover:text-white hover:bg-slate-800 rounded-xl"><Edit3 size={16}/></button>
                      <button onClick={() => handleDelete(item.id)} className="p-2.5 text-slate-600 hover:text-rose-500 hover:bg-rose-500/10 rounded-xl"><Trash2 size={16}/></button>
                    </div>
                  </div>
                  <div className="grid grid-cols-2 gap-x-4 gap-y-2 mb-4">
                    <div className="flex items-center gap-1.5 text-[9px] font-bold text-slate-500 uppercase"><FileText size={12} className="text-emerald-500" /> Drawing: {item.drawingRef || '-'}</div>
                    <div className="flex items-center gap-1.5 text-[9px] font-bold text-slate-500 uppercase"><MapPin size={12} className="text-yellow-500" /> Frame: {item.frame}</div>
                    <div className="flex items-center gap-1.5 text-[9px] font-bold text-slate-500 uppercase"><ShieldCheck size={12} className="text-orange-500" /> Stranger: {item.stranger}</div>
                  </div>
                </div>

                <div className="flex items-center justify-between pt-4 border-t border-slate-800/50">
                  <div className="flex items-center gap-2">
                    <div className="w-6 h-6 rounded-full bg-slate-800 flex items-center justify-center text-[8px] font-black text-slate-500">
                      {item.inputByNIK?.slice(-2)}
                    </div>
                    <span className="text-[8px] font-black text-slate-600 uppercase">Tech: {item.inputByNIK}</span>
                  </div>
                  <button 
                    onClick={() => toggleStatus(item.id, item.status)}
                    className={`flex items-center gap-2 px-6 py-2.5 rounded-xl text-[9px] font-black uppercase tracking-widest transition-all ${
                      item.status === 'Terpasang' ? 'bg-emerald-500 text-slate-950 shadow-emerald-500/20' : 'bg-slate-900 text-slate-500 border border-slate-800'
                    }`}
                  >
                    {item.status === 'Terpasang' ? <CheckCircle2 size={14}/> : <Clock size={14}/>}
                    {item.status}
                  </button>
                </div>
              </div>
            </div>
          ))}
        </div>
      </main>

      <div className="fixed bottom-8 right-8 z-[100]">
        <button onClick={() => { setEditingItem(null); setIsModalOpen(true); }} className="w-16 h-16 bg-gradient-to-br from-yellow-400 to-orange-500 rounded-[1.75rem] flex items-center justify-center shadow-2xl active:scale-90 transition-all group overflow-hidden">
          <Plus size={32} className="text-slate-950 group-hover:rotate-90 transition-transform" />
        </button>
      </div>

      {isModalOpen && (
        <div className="fixed inset-0 z-[200] flex items-center justify-center p-4 bg-[#020617]/95 backdrop-blur-xl animate-in fade-in">
          <div className="bg-[#0f172a] w-full max-w-lg rounded-[3rem] border border-slate-800 shadow-3xl max-h-[90vh] overflow-y-auto no-scrollbar relative">
            <div className="sticky top-0 bg-[#0f172a]/90 backdrop-blur-md p-6 border-b border-slate-800 flex justify-between items-center z-10">
              <div>
                <h2 className="font-black text-xl text-white uppercase leading-none">{editingItem ? 'Update Entry' : 'New Entry'}</h2>
                <p className="text-[8px] font-black text-slate-500 uppercase mt-1">Layout Documentation System</p>
              </div>
              <button onClick={() => setIsModalOpen(false)} className="text-slate-500 hover:text-white bg-slate-900 p-3 rounded-2xl"><X size={20}/></button>
            </div>
            
            <form onSubmit={handleSaveItem} className="p-8 space-y-6">
              <div onClick={() => fileInputRef.current.click()} className="relative h-48 rounded-[2.5rem] border-2 border-dashed border-slate-800 hover:border-yellow-500/50 transition-all flex flex-col items-center justify-center cursor-pointer overflow-hidden bg-slate-950 shadow-inner group">
                {newItem.image ? (
                  <img src={newItem.image} className="w-full h-full object-cover brightness-75 group-hover:brightness-100" alt="Upload" />
                ) : (
                  <>
                    <Camera className="text-yellow-500 mb-3" size={32} />
                    <span className="text-[10px] font-black text-slate-500 uppercase tracking-widest text-center px-4">Tap to capture component photo</span>
                  </>
                )}
                <input ref={fileInputRef} type="file" accept="image/*" onChange={processImage} className="hidden" />
              </div>

              <div className="grid grid-cols-1 sm:grid-cols-2 gap-5">
                <div className="sm:col-span-2">
                  <label className="text-[9px] font-black text-slate-500 uppercase ml-4 mb-2 block">Component Name</label>
                  <input required className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-4 px-6 text-white font-bold outline-none focus:ring-2 focus:ring-yellow-500/30" placeholder="E.G. GPS ANTENNA" value={newItem.name} onChange={e => setNewItem({...newItem, name: e.target.value.toUpperCase()})} />
                </div>
                <div>
                  <label className="text-[9px] font-black text-slate-500 uppercase ml-4 mb-2 block">Drawing Reference</label>
                  <input required className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-4 px-6 text-white font-mono font-bold text-xs outline-none focus:ring-2 focus:ring-yellow-500/30" placeholder="DWG-212-001" value={newItem.drawingRef} onChange={e => setNewItem({...newItem, drawingRef: e.target.value.toUpperCase()})} />
                </div>
                <div>
                  <label className="text-[9px] font-black text-slate-500 uppercase ml-4 mb-2 block">Archive Folder</label>
                  <select className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-4 px-6 text-white font-black uppercase text-[10px] outline-none cursor-pointer" value={currentFolderId || newItem.folderId} onChange={e => setNewItem({...newItem, folderId: e.target.value})} disabled={!!currentFolderId}>
                    <option value="">Root Archive</option>
                    {folders.map(f => ( <option key={f.id} value={f.id}>{f.name}</option> ))}
                  </select>
                </div>
                <div>
                  <label className="text-[9px] font-black text-slate-500 uppercase ml-4 mb-2 block">Aircraft Type</label>
                  <select className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-4 px-6 text-white font-black uppercase text-[10px] outline-none cursor-pointer" value={newItem.plane} onChange={e => setNewItem({...newItem, plane: e.target.value})}>
                    <option value="NC212">NC212</option>
                    <option value="CN235">CN235</option>
                  </select>
                </div>
                <div>
                  <label className="text-[9px] font-black text-slate-500 uppercase ml-4 mb-2 block">Frame Location</label>
                  <input required className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-4 px-6 text-white font-bold outline-none focus:ring-2 focus:ring-yellow-500/30" placeholder="FR-12" value={newItem.frame} onChange={e => setNewItem({...newItem, frame: e.target.value.toUpperCase()})} />
                </div>
                <div className="sm:col-span-2">
                  <label className="text-[9px] font-black text-slate-500 uppercase ml-4 mb-2 block">Stranger Location</label>
                  <input required className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-4 px-6 text-white font-bold outline-none border-orange-500/20 focus:ring-2 focus:ring-orange-500/30" placeholder="STR-5" value={newItem.stranger} onChange={e => setNewItem({...newItem, stranger: e.target.value.toUpperCase()})} />
                </div>
              </div>

              <button disabled={submitting} className="w-full bg-gradient-to-r from-yellow-500 to-orange-500 text-slate-950 py-6 rounded-3xl font-black uppercase tracking-[0.2em] shadow-xl shadow-yellow-500/20 active:scale-[0.97] transition-all flex items-center justify-center gap-3">
                {submitting ? <Loader2 className="animate-spin" size={20}/> : <Database size={20}/>}
                {editingItem ? 'Sync Updates' : 'Authorize Entry'}
              </button>
            </form>
          </div>
        </div>
      )}

      {isFolderModalOpen && (
        <div className="fixed inset-0 z-[250] flex items-center justify-center p-4 bg-[#020617]/90 backdrop-blur-sm animate-in fade-in">
          <div className="bg-[#0f172a] w-full max-w-xs rounded-[2.5rem] border border-slate-800 p-8 shadow-3xl">
            <h2 className="text-xl font-black uppercase text-yellow-500 mb-6">Create Archive</h2>
            <form onSubmit={handleAddFolder} className="space-y-4">
              <input autoFocus required className="w-full bg-slate-950 border border-slate-800 rounded-2xl py-5 px-6 text-white font-bold outline-none text-center focus:ring-2 focus:ring-yellow-500/30" placeholder="FOLDER NAME" value={newFolderName} onChange={e => setNewFolderName(e.target.value)} />
              <div className="flex gap-2">
                <button type="button" onClick={() => setIsFolderModalOpen(false)} className="flex-1 py-4 bg-slate-900 text-slate-500 rounded-2xl font-black uppercase text-[9px]">Cancel</button>
                <button type="submit" disabled={submitting} className="flex-[2] py-4 bg-yellow-500 text-slate-950 rounded-2xl font-black uppercase text-[9px] shadow-lg">Build</button>
              </div>
            </form>
          </div>
        </div>
      )}

      {previewImage && (
        <div className="fixed inset-0 z-[300] bg-slate-950/95 flex items-center justify-center p-4 animate-in fade-in" onClick={() => setPreviewImage(null)}>
          <button className="absolute top-8 right-8 text-white bg-slate-900 p-4 rounded-full hover:bg-slate-800 transition-colors"><X size={32}/></button>
          <img src={previewImage} className="max-w-full max-h-[90vh] object-contain rounded-3xl shadow-3xl border border-white/10" alt="Full Preview" />
        </div>
      )}
    </div>
  );
};

export default App;

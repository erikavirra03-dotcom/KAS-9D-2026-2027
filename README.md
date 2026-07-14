# KAS-9D-2026-2027
KAS
import React, { useState, useEffect, useMemo } from 'react';
import { Lock, Unlock, LogOut, Plus, Pencil, Trash2, X, Check, TrendingUp, TrendingDown, Wallet, ChevronDown, AlertCircle, Loader2 } from 'lucide-react';

const OPERATOR_PASSWORD = '9DINASTIERIKA'; // Ganti kata sandi di sini

const INK = {
  paper: '#FBF7EE',
  paperLine: '#DED2B0',
  ink: '#2B3A55',
  inkFaint: '#6B7280',
  green: '#2F6F4E',
  greenBg: '#E7F0EA',
  red: '#A6322C',
  redBg: '#F6E7E5',
  gold: '#B08D57',
  goldBg: '#F3ECDD',
};

const fmtRupiah = (n) =>
  'Rp' + Number(n || 0).toLocaleString('id-ID', { maximumFractionDigits: 0 });

const fmtTanggal = (iso) => {
  if (!iso) return '';
  const d = new Date(iso + 'T00:00:00');
  return d.toLocaleDateString('id-ID', { day: '2-digit', month: 'short', year: 'numeric' });
};

const monthKey = (iso) => iso ? iso.slice(0, 7) : '';
const monthLabel = (key) => {
  const [y, m] = key.split('-');
  const d = new Date(Number(y), Number(m) - 1, 1);
  return d.toLocaleDateString('id-ID', { month: 'long', year: 'numeric' });
};

const emptyForm = () => ({
  id: null,
  date: new Date().toISOString().slice(0, 10),
  type: 'masuk',
  category: '',
  description: '',
  amount: '',
});

export default function PerbendaharaanKelas() {
  const [loading, setLoading] = useState(true);
  const [saving, setSaving] = useState(false);
  const [className, setClassName] = useState('Kas Kelas');
  const [transactions, setTransactions] = useState([]);
  const [mode, setMode] = useState('visitor'); // visitor | operator

  const [showLogin, setShowLogin] = useState(false);
  const [pwInput, setPwInput] = useState('');
  const [pwError, setPwError] = useState('');

  const [editingTitle, setEditingTitle] = useState(false);
  const [titleInput, setTitleInput] = useState('');

  const [showForm, setShowForm] = useState(false);
  const [form, setForm] = useState(emptyForm());
  const [formError, setFormError] = useState('');

  const [monthFilter, setMonthFilter] = useState('all');
  const [confirmDeleteId, setConfirmDeleteId] = useState(null);
  const [loadError, setLoadError] = useState('');

  useEffect(() => {
    (async () => {
      try {
        const res = await window.storage.get('kas-data', true);
        if (res && res.value) {
          const parsed = JSON.parse(res.value);
          setClassName(parsed.className || 'Kas Kelas');
          setTransactions(Array.isArray(parsed.transactions) ? parsed.transactions : []);
        }
      } catch (e) {
        // key belum ada — mulai dengan data kosong
      } finally {
        setLoading(false);
      }
    })();
  }, []);

  const persist = async (next) => {
    setSaving(true);
    setLoadError('');
    try {
      const payload = {
        className: next.className !== undefined ? next.className : className,
        transactions: next.transactions !== undefined ? next.transactions : transactions,
      };
      const result = await window.storage.set('kas-data', JSON.stringify(payload), true);
      if (!result) throw new Error('gagal menyimpan');
      if (next.className !== undefined) setClassName(next.className);
      if (next.transactions !== undefined) setTransactions(next.transactions);
    } catch (e) {
      setLoadError('Gagal menyimpan data. Periksa koneksi lalu coba lagi.');
    } finally {
      setSaving(false);
    }
  };

  const handleLogin = () => {
    if (pwInput === OPERATOR_PASSWORD) {
      setMode('operator');
      setShowLogin(false);
      setPwInput('');
      setPwError('');
    } else {
      setPwError('Kata sandi salah. Coba lagi.');
    }
  };

  const handleLogout = () => {
    setMode('visitor');
    setShowForm(false);
    setEditingTitle(false);
    setConfirmDeleteId(null);
  };

  const openAdd = () => {
    setForm(emptyForm());
    setFormError('');
    setShowForm(true);
  };

  const openEdit = (t) => {
    setForm({ ...t, amount: String(t.amount) });
    setFormError('');
    setShowForm(true);
  };

  const submitForm = () => {
    if (!form.date) return setFormError('Tanggal wajib diisi.');
    if (!form.description.trim()) return setFormError('Keterangan wajib diisi.');
    const amt = Number(form.amount);
    if (!amt || amt <= 0) return setFormError('Jumlah harus lebih dari 0.');

    let next;
    if (form.id) {
      next = transactions.map((t) =>
        t.id === form.id ? { ...t, ...form, amount: amt } : t
      );
    } else {
      const newT = {
        ...form,
        id: Date.now().toString(36) + Math.random().toString(36).slice(2, 7),
        amount: amt,
        createdAt: Date.now(),
      };
      next = [...transactions, newT];
    }
    persist({ transactions: next });
    setShowForm(false);
  };

  const deleteTransaction = (id) => {
    const next = transactions.filter((t) => t.id !== id);
    persist({ transactions: next });
    setConfirmDeleteId(null);
  };

  const saveTitle = () => {
    const t = titleInput.trim() || 'Kas Kelas';
    persist({ className: t });
    setEditingTitle(false);
  };

  const withBalance = useMemo(() => {
    const asc = [...transactions].sort(
      (a, b) => a.date.localeCompare(b.date) || (a.createdAt || 0) - (b.createdAt || 0)
    );
    let running = 0;
    return asc.map((t) => {
      running += t.type === 'masuk' ? Number(t.amount) : -Number(t.amount);
      return { ...t, balance: running };
    });
  }, [transactions]);

  const totals = useMemo(() => {
    let masuk = 0, keluar = 0;
    transactions.forEach((t) => {
      if (t.type === 'masuk') masuk += Number(t.amount);
      else keluar += Number(t.amount);
    });
    return { masuk, keluar, saldo: masuk - keluar };
  }, [transactions]);

  const months = useMemo(() => {
    const set = new Set(transactions.map((t) => monthKey(t.date)));
    return Array.from(set).sort().reverse();
  }, [transactions]);

  const displayList = useMemo(() => {
    const filtered = monthFilter === 'all'
      ? withBalance
      : withBalance.filter((t) => monthKey(t.date) === monthFilter);
    return [...filtered].reverse();
  }, [withBalance, monthFilter]);

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center" style={{ background: INK.paper }}>
        <div className="flex flex-col items-center gap-3">
          <Loader2 className="animate-spin" size={28} style={{ color: INK.ink }} />
          <p className="text-sm" style={{ color: INK.inkFaint, fontFamily: 'Plus Jakarta Sans, sans-serif' }}>
            Membuka buku kas...
          </p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen pb-10" style={{ background: INK.paper }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Lora:wght@500;600;700&family=Plus+Jakarta+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@500;600&display=swap');
        .font-display { font-family: 'Lora', serif; }
        .font-body { font-family: 'Plus Jakarta Sans', sans-serif; }
        .font-mono { font-family: 'IBM Plex Mono', monospace; font-variant-numeric: tabular-nums; }
      `}</style>

      {/* Header / ledger title bar */}
      <div className="px-5 pt-6 pb-4" style={{ borderBottom: `3px double ${INK.ink}` }}>
        <div className="flex items-start justify-between gap-3">
          <div className="flex-1 min-w-0">
            {editingTitle ? (
              <div className="flex items-center gap-2">
                <input
                  autoFocus
                  value={titleInput}
                  onChange={(e) => setTitleInput(e.target.value)}
                  className="font-display text-xl font-semibold bg-transparent border-b-2 outline-none flex-1 min-w-0"
                  style={{ color: INK.ink, borderColor: INK.gold }}
                />
                <button onClick={saveTitle} className="p-1.5 rounded-full" style={{ background: INK.green }}>
                  <Check size={16} color="white" />
                </button>
                <button onClick={() => setEditingTitle(false)} className="p-1.5 rounded-full" style={{ background: INK.inkFaint }}>
                  <X size={16} color="white" />
                </button>
              </div>
            ) : (
              <div className="flex items-center gap-2">
                <h1 className="font-display text-xl font-semibold truncate" style={{ color: INK.ink }}>
                  {className}
                </h1>
                {mode === 'operator' && (
                  <button onClick={() => { setTitleInput(className); setEditingTitle(true); }} className="opacity-60 shrink-0">
                    <Pencil size={14} style={{ color: INK.ink }} />
                  </button>
                )}
              </div>
            )}
            <p className="font-body text-xs mt-0.5" style={{ color: INK.inkFaint }}>
              Perbendaharaan Kelas · Buku Kas
            </p>
          </div>

          {mode === 'visitor' ? (
            <button
              onClick={() => setShowLogin(true)}
              className="font-body text-xs font-semibold flex items-center gap-1.5 px-3 py-2 rounded-full shrink-0"
              style={{ background: INK.ink, color: INK.paper }}
            >
              <Lock size={13} /> Operator
            </button>
          ) : (
            <div className="flex flex-col items-end gap-1.5 shrink-0">
              <div
                className="font-body text-[10px] font-bold px-2.5 py-1 rounded-full border-2 flex items-center gap-1"
                style={{ borderColor: INK.gold, color: INK.gold, transform: 'rotate(-3deg)' }}
              >
                <Unlock size={11} /> OPERATOR AKTIF
              </div>
              <button onClick={handleLogout} className="font-body text-[11px] flex items-center gap-1" style={{ color: INK.inkFaint }}>
                <LogOut size={12} /> Keluar
              </button>
            </div>
          )}
        </div>
      </div>

      {/* Saldo card */}
      <div className="px-5 pt-5">
        <p className="font-body text-xs uppercase tracking-wide" style={{ color: INK.inkFaint }}>Saldo Saat Ini</p>
        <p className="font-mono text-3xl font-semibold mt-1" style={{ color: totals.saldo < 0 ? INK.red : INK.ink }}>
          {fmtRupiah(totals.saldo)}
        </p>
        <div className="flex gap-3 mt-3">
          <div className="flex-1 rounded-xl px-3 py-2.5 flex items-center gap-2" style={{ background: INK.greenBg }}>
            <TrendingUp size={16} style={{ color: INK.green }} />
            <div>
              <p className="font-body text-[10px]" style={{ color: INK.green }}>Total Masuk</p>
              <p className="font-mono text-sm font-semibold" style={{ color: INK.green }}>{fmtRupiah(totals.masuk)}</p>
            </div>
          </div>
          <div className="flex-1 rounded-xl px-3 py-2.5 flex items-center gap-2" style={{ background: INK.redBg }}>
            <TrendingDown size={16} style={{ color: INK.red }} />
            <div>
              <p className="font-body text-[10px]" style={{ color: INK.red }}>Total Keluar</p>
              <p className="font-mono text-sm font-semibold" style={{ color: INK.red }}>{fmtRupiah(totals.keluar)}</p>
            </div>
          </div>
        </div>
      </div>

      {loadError && (
        <div className="mx-5 mt-4 rounded-lg px-3 py-2 flex items-center gap-2" style={{ background: INK.redBg }}>
          <AlertCircle size={14} style={{ color: INK.red }} />
          <p className="font-body text-xs" style={{ color: INK.red }}>{loadError}</p>
        </div>
      )}

      {/* Action row */}
      {mode === 'operator' && (
        <div className="px-5 mt-5">
          <button
            onClick={openAdd}
            className="w-full font-body text-sm font-semibold flex items-center justify-center gap-2 py-3 rounded-xl"
            style={{ background: INK.ink, color: INK.paper }}
          >
            <Plus size={16} /> Tambah Transaksi
          </button>
        </div>
      )}

      {/* Month filter */}
      {months.length > 0 && (
        <div className="px-5 mt-5">
          <div className="relative inline-block w-full">
            <select
              value={monthFilter}
              onChange={(e) => setMonthFilter(e.target.value)}
              className="font-body text-xs w-full appearance-none rounded-lg px-3 py-2 pr-8 border"
              style={{ borderColor: INK.paperLine, color: INK.ink, background: 'white' }}
            >
              <option value="all">Semua Bulan</option>
              {months.map((m) => (
                <option key={m} value={m}>{monthLabel(m)}</option>
              ))}
            </select>
            <ChevronDown size={14} className="absolute right-2.5 top-2.5 pointer-events-none" style={{ color: INK.inkFaint }} />
          </div>
        </div>
      )}

      {/* Ledger list */}
      <div className="px-5 mt-4">
        {displayList.length === 0 ? (
          <div className="text-center py-14">
            <Wallet size={30} className="mx-auto mb-2" style={{ color: INK.paperLine }} />
            <p className="font-body text-sm" style={{ color: INK.inkFaint }}>
              {transactions.length === 0
                ? 'Belum ada transaksi tercatat.'
                : 'Tidak ada transaksi pada bulan ini.'}
            </p>
            {mode === 'visitor' && transactions.length === 0 && (
              <p className="font-body text-xs mt-1" style={{ color: INK.inkFaint }}>
                Masuk sebagai operator untuk mulai mencatat.
              </p>
            )}
          </div>
        ) : (
          <div>
            {displayList.map((t, i) => (
              <div key={t.id}>
                <div className="py-3 flex items-start justify-between gap-2">
                  <div className="min-w-0 flex-1">
                    <div className="flex items-center gap-2 flex-wrap">
                      <span className="font-body text-[11px]" style={{ color: INK.inkFaint }}>{fmtTanggal(t.date)}</span>
                      {t.category && (
                        <span
                          className="font-body text-[10px] px-1.5 py-0.5 rounded"
                          style={{ background: INK.goldBg, color: INK.gold }}
                        >
                          {t.category}
                        </span>
                      )}
                    </div>
                    <p className="font-body text-sm mt-0.5 truncate" style={{ color: INK.ink }}>{t.description}</p>
                    <p className="font-mono text-[10px] mt-0.5" style={{ color: INK.inkFaint }}>
                      Saldo: {fmtRupiah(t.balance)}
                    </p>
                  </div>
                  <div className="text-right shrink-0">
                    <p className="font-mono text-sm font-semibold" style={{ color: t.type === 'masuk' ? INK.green : INK.red }}>
                      {t.type === 'masuk' ? '+' : '-'}{fmtRupiah(t.amount)}
                    </p>
                    {mode === 'operator' && (
                      <div className="flex items-center gap-2 justify-end mt-1.5">
                        <button onClick={() => openEdit(t)}>
                          <Pencil size={13} style={{ color: INK.inkFaint }} />
                        </button>
                        {confirmDeleteId === t.id ? (
                          <div className="flex items-center gap-1">
                            <button onClick={() => deleteTransaction(t.id)} className="font-body text-[10px] font-semibold" style={{ color: INK.red }}>Ya</button>
                            <button onClick={() => setConfirmDeleteId(null)} className="font-body text-[10px]" style={{ color: INK.inkFaint }}>Batal</button>
                          </div>
                        ) : (
                          <button onClick={() => setConfirmDeleteId(t.id)}>
                            <Trash2 size={13} style={{ color: INK.inkFaint }} />
                          </button>
                        )}
                      </div>
                    )}
                  </div>
                </div>
                {i < displayList.length - 1 && (
                  <div style={{ borderTop: `1px dashed ${INK.paperLine}` }} />
                )}
              </div>
            ))}
          </div>
        )}
      </div>

      {/* Login modal */}
      {showLogin && (
        <div className="fixed inset-0 flex items-center justify-center px-6 z-50" style={{ background: 'rgba(43,58,85,0.45)' }}>
          <div className="w-full max-w-xs rounded-2xl p-5" style={{ background: INK.paper, border: `2px solid ${INK.ink}` }}>
            <div className="flex items-center justify-between mb-3">
              <h2 className="font-display text-base font-semibold flex items-center gap-2" style={{ color: INK.ink }}>
                <Lock size={16} /> Masuk Operator
              </h2>
              <button onClick={() => { setShowLogin(false); setPwInput(''); setPwError(''); }}>
                <X size={16} style={{ color: INK.inkFaint }} />
              </button>
            </div>
            <input
              type="password"
              autoFocus
              value={pwInput}
              onChange={(e) => { setPwInput(e.target.value); setPwError(''); }}
              onKeyDown={(e) => e.key === 'Enter' && handleLogin()}
              placeholder="Kata sandi"
              className="font-body text-sm w-full rounded-lg px-3 py-2.5 border outline-none"
              style={{ borderColor: pwError ? INK.red : INK.paperLine }}
            />
            {pwError && <p className="font-body text-xs mt-1.5" style={{ color: INK.red }}>{pwError}</p>}
            <button
              onClick={handleLogin}
              className="font-body text-sm font-semibold w-full mt-4 py-2.5 rounded-lg"
              style={{ background: INK.ink, color: INK.paper }}
            >
              Buka Buku Kas
            </button>
          </div>
        </div>
      )}

      {/* Add/Edit form modal */}
      {showForm && (
        <div className="fixed inset-0 flex items-end sm:items-center justify-center z-50" style={{ background: 'rgba(43,58,85,0.45)' }}>
          <div className="w-full sm:max-w-sm rounded-t-2xl sm:rounded-2xl p-5 max-h-[85vh] overflow-y-auto" style={{ background: INK.paper, border: `2px solid ${INK.ink}` }}>
            <div className="flex items-center justify-between mb-4">
              <h2 className="font-display text-base font-semibold" style={{ color: INK.ink }}>
                {form.id ? 'Ubah Transaksi' : 'Tambah Transaksi'}
              </h2>
              <button onClick={() => setShowForm(false)}>
                <X size={16} style={{ color: INK.inkFaint }} />
              </button>
            </div>

            <div className="flex gap-2 mb-4">
              <button
                onClick={() => setForm((f) => ({ ...f, type: 'masuk' }))}
                className="flex-1 font-body text-sm font-semibold py-2 rounded-lg border-2"
                style={form.type === 'masuk'
                  ? { background: INK.greenBg, borderColor: INK.green, color: INK.green }
                  : { borderColor: INK.paperLine, color: INK.inkFaint }}
              >
                Masuk
              </button>
              <button
                onClick={() => setForm((f) => ({ ...f, type: 'keluar' }))}
                className="flex-1 font-body text-sm font-semibold py-2 rounded-lg border-2"
                style={form.type === 'keluar'
                  ? { background: INK.redBg, borderColor: INK.red, color: INK.red }
                  : { borderColor: INK.paperLine, color: INK.inkFaint }}
              >
                Keluar
              </button>
            </div>

            <label className="font-body text-xs" style={{ color: INK.inkFaint }}>Tanggal</label>
            <input
              type="date"
              value={form.date}
              onChange={(e) => setForm((f) => ({ ...f, date: e.target.value }))}
              className="font-body text-sm w-full rounded-lg px-3 py-2.5 border outline-none mt-1 mb-3"
              style={{ borderColor: INK.paperLine }}
            />

            <label className="font-body text-xs" style={{ color: INK.inkFaint }}>Kategori (opsional)</label>
            <input
              type="text"
              value={form.category}
              onChange={(e) => setForm((f) => ({ ...f, category: e.target.value }))}
              placeholder="Iuran, Konsumsi, ATK, ..."
              className="font-body text-sm w-full rounded-lg px-3 py-2.5 border outline-none mt-1 mb-3"
              style={{ borderColor: INK.paperLine }}
            />

            <label className="font-body text-xs" style={{ color: INK.inkFaint }}>Keterangan</label>
            <input
              type="text"
              value={form.description}
              onChange={(e) => setForm((f) => ({ ...f, description: e.target.value }))}
              placeholder="Contoh: Iuran kas minggu ini"
              className="font-body text-sm w-full rounded-lg px-3 py-2.5 border outline-none mt-1 mb-3"
              style={{ borderColor: INK.paperLine }}
            />

            <label className="font-body text-xs" style={{ color: INK.inkFaint }}>Jumlah (Rp)</label>
            <input
              type="number"
              inputMode="numeric"
              value={form.amount}
              onChange={(e) => setForm((f) => ({ ...f, amount: e.target.value }))}
              placeholder="0"
              className="font-mono text-sm w-full rounded-lg px-3 py-2.5 border outline-none mt-1"
              style={{ borderColor: INK.paperLine }}
            />

            {formError && (
              <p className="font-body text-xs mt-2 flex items-center gap-1" style={{ color: INK.red }}>
                <AlertCircle size={12} /> {formError}
              </p>
            )}

            <button
              onClick={submitForm}
              disabled={saving}
              className="font-body text-sm font-semibold w-full mt-4 py-2.5 rounded-lg flex items-center justify-center gap-2"
              style={{ background: INK.ink, color: INK.paper, opacity: saving ? 0.6 : 1 }}
            >
              {saving ? <Loader2 size={15} className="animate-spin" /> : <Check size={15} />}
              {form.id ? 'Simpan Perubahan' : 'Simpan Transaksi'}
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

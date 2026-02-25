[index.html](https://github.com/user-attachments/files/25541424/index.html)
<!DOCTYPE html>
<html lang="ko">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Expense Tracker - 가계부</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Google Fonts: Inter & Outfit -->
    <link
        href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=Outfit:wght@600;800&display=swap"
        rel="stylesheet">
    <!-- React & Babel CDN -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    <!-- Chart.js CDN -->
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root {
            --bg-dark: #0f172a;
            --card-dark: #1e293b;
        }

        body {
            font-family: 'Inter', sans-serif;
            background-color: var(--bg-dark);
            color: #f8fafc;
            transition: background-color 0.3s ease;
        }

        .font-outfit {
            font-family: 'Outfit', sans-serif;
        }

        .glass {
            background: rgba(30, 41, 59, 0.7);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.1);
        }

        .custom-scrollbar::-webkit-scrollbar {
            width: 6px;
        }

        .custom-scrollbar::-webkit-scrollbar-track {
            background: transparent;
        }

        .custom-scrollbar::-webkit-scrollbar-thumb {
            background: rgba(255, 255, 255, 0.1);
            border-radius: 10px;
        }

        .animate-in {
            animation: fadeIn 0.4s ease-out;
        }

        @keyframes fadeIn {
            from {
                opacity: 0;
                transform: translateY(10px);
            }

            to {
                opacity: 1;
                transform: translateY(0);
            }
        }
    </style>
</head>

<body>
    <div id="root"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef, useMemo } = React;

        const CATEGORIES = [
            { id: 'food', label: '식비', icon: '🍔', color: '#f87171' },
            { id: 'transport', label: '교통', icon: '🚌', color: '#fb923c' },
            { id: 'shopping', label: '쇼핑', icon: '🛍️', color: '#fbbf24' },
            { id: 'housing', label: '주거', icon: '🏠', color: '#4ade80' },
            { id: 'entertainment', label: '취미', icon: '🎮', color: '#2dd4bf' },
            { id: 'health', label: '건강', icon: '🏥', color: '#60a5fa' },
            { id: 'etc', label: '기타', icon: '✨', color: '#a78bfa' }
        ];

        const ExpenseApp = () => {
            const [expenses, setExpenses] = useState(() => {
                const saved = localStorage.getItem('expenses_data');
                return saved ? JSON.parse(saved) : [];
            });
            const [isDarkMode, setIsDarkMode] = useState(true);
            const [formData, setFormData] = useState({
                id: null,
                category: 'food',
                amount: '',
                date: new Date().toISOString().split('T')[0],
                memo: ''
            });
            const [filterCategory, setFilterCategory] = useState('all');
            const chartRef = useRef(null);
            const chartInstance = useRef(null);

            // Persistence
            useEffect(() => {
                localStorage.setItem('expenses_data', JSON.stringify(expenses));
                updateChart();
            }, [expenses]);

            // Chart Initialization/Update
            const updateChart = () => {
                if (!chartRef.current) return;

                const categoryTotals = CATEGORIES.map(cat => {
                    return expenses
                        .filter(e => e.category === cat.id)
                        .reduce((sum, e) => sum + Number(e.amount), 0);
                });

                if (chartInstance.current) {
                    chartInstance.current.destroy();
                }

                const ctx = chartRef.current.getContext('2d');
                chartInstance.current = new Chart(ctx, {
                    type: 'doughnut',
                    data: {
                        labels: CATEGORIES.map(c => c.label),
                        datasets: [{
                            data: categoryTotals,
                            backgroundColor: CATEGORIES.map(c => c.color),
                            borderWidth: 0,
                            hoverOffset: 10
                        }]
                    },
                    options: {
                        responsive: true,
                        maintainAspectRatio: false,
                        plugins: {
                            legend: {
                                position: 'bottom',
                                labels: { color: '#94a3b8', font: { size: 10 } }
                            }
                        },
                        cutout: '70%'
                    }
                });
            };

            useEffect(() => {
                updateChart();
            }, []);

            const handleSubmit = (e) => {
                e.preventDefault();
                if (!formData.amount || formData.amount <= 0) return;

                if (formData.id) {
                    setExpenses(prev => prev.map(ex => ex.id === formData.id ? { ...formData } : ex));
                } else {
                    setExpenses(prev => [{ ...formData, id: Date.now() }, ...prev]);
                }

                setFormData({
                    id: null,
                    category: 'food',
                    amount: '',
                    date: new Date().toISOString().split('T')[0],
                    memo: ''
                });
            };

            const handleEdit = (ex) => {
                setFormData({ ...ex });
                window.scrollTo({ top: 0, behavior: 'smooth' });
            };

            const handleDelete = (id) => {
                if (confirm('정말 삭제하시겠습니까?')) {
                    setExpenses(prev => prev.filter(ex => ex.id !== id));
                }
            };

            const filteredExpenses = useMemo(() => {
                return expenses.filter(ex => filterCategory === 'all' || ex.category === filterCategory);
            }, [expenses, filterCategory]);

            const totalAmount = useMemo(() => {
                return expenses.reduce((sum, ex) => sum + Number(ex.amount), 0);
            }, [expenses]);

            const formatCurrency = (val) => {
                return new Intl.NumberFormat('ko-KR', { style: 'currency', currency: 'KRW' }).format(val);
            };

            const exportToCSV = () => {
                const headers = ['날짜,카테고리,금액,메모\n'];
                const rows = expenses.map(ex => {
                    const catLabel = CATEGORIES.find(c => c.id === ex.category)?.label || '';
                    return `${ex.date},${catLabel},${ex.amount},"${ex.memo}"\n`;
                });
                const blob = new Blob(['\ufeff' + headers + rows.join('')], { type: 'text/csv;charset=utf-8;' });
                const link = document.createElement('a');
                link.href = URL.createObjectURL(blob);
                link.download = `지출내역_${new Date().toLocaleDateString()}.csv`;
                link.click();
            };

            return (
                <div className={`min-h-screen transition-colors duration-300 ${isDarkMode ? 'bg-[#0f172a] text-slate-100' : 'bg-slate-50 text-slate-900'}`}>
                    {/* Header */}
                    <header className={`sticky top-0 z-10 border-b p-4 mb-4 ${isDarkMode ? 'bg-[#0f172a]/80 border-slate-800' : 'bg-white/80 border-slate-200'} backdrop-blur-md`}>
                        <div className="max-w-6xl mx-auto flex justify-between items-center">
                            <h1 className="text-2xl font-outfit font-extrabold bg-gradient-to-r from-blue-400 to-indigo-500 bg-clip-text text-transparent italic">
                                ExpenseTracker
                            </h1>
                            <div className="flex gap-4">
                                <button onClick={exportToCSV} className="text-xs font-medium px-3 py-1 rounded-full border border-blue-500/30 text-blue-400 hover:bg-blue-500/10 transition-colors">
                                    CSV 내보내기
                                </button>
                                <button
                                    onClick={() => setIsDarkMode(!isDarkMode)}
                                    className="p-2 rounded-xl border border-white/10 glass"
                                >
                                    {isDarkMode ? '☀️' : '🌙'}
                                </button>
                            </div>
                        </div>
                    </header>

                    <main className="max-w-6xl mx-auto p-4 grid gap-8 lg:grid-cols-12">
                        {/* Left: Input & Chart */}
                        <div className="lg:col-span-5 space-y-6">
                            {/* Summary Card */}
                            <div className={`rounded-3xl p-8 glass overflow-hidden relative border border-white/5`}>
                                <div className="relative z-10 flex justify-between items-start mb-4">
                                    <div>
                                        <p className="text-slate-400 text-sm font-medium mb-1">총 지출</p>
                                        <h2 className="text-4xl font-outfit font-black tracking-tight tracking-wide">
                                            {formatCurrency(totalAmount)}
                                        </h2>
                                    </div>
                                    <span className="p-3 bg-blue-500/20 rounded-2xl text-2xl">💰</span>
                                </div>
                                <div className="h-64 relative">
                                    <canvas ref={chartRef}></canvas>
                                    <div className="absolute inset-0 flex flex-col items-center justify-center pointer-events-none">
                                        <span className="text-slate-500 text-xs italic">CATEGORY MIX</span>
                                    </div>
                                </div>
                            </div>

                            {/* Input Form */}
                            <div className={`rounded-3xl p-6 ${isDarkMode ? 'bg-slate-800/50' : 'bg-white shadow-xl'} border border-white/5`}>
                                <h3 className="text-lg font-bold mb-4 flex items-center gap-2">
                                    <span className="w-2 h-6 bg-blue-500 rounded-full"></span>
                                    {formData.id ? '지출 내역 수정' : '새 지출 등록'}
                                </h3>
                                <form onSubmit={handleSubmit} className="space-y-4">
                                    <div className="grid grid-cols-2 gap-4">
                                        <div className="space-y-1">
                                            <label className="text-xs font-semibold text-slate-500">날짜</label>
                                            <input
                                                type="date"
                                                value={formData.date}
                                                onChange={e => setFormData({ ...formData, date: e.target.value })}
                                                className={`w-full p-3 rounded-xl outline-none border transition-all ${isDarkMode ? 'bg-slate-900 border-slate-700 focus:border-blue-500' : 'bg-slate-100 border-slate-200 focus:border-blue-500'}`}
                                            />
                                        </div>
                                        <div className="space-y-1">
                                            <label className="text-xs font-semibold text-slate-500">카테고리</label>
                                            <select
                                                value={formData.category}
                                                onChange={e => setFormData({ ...formData, category: e.target.value })}
                                                className={`w-full p-3 rounded-xl outline-none border transition-all appearance-none ${isDarkMode ? 'bg-slate-900 border-slate-700 focus:border-blue-500' : 'bg-slate-100 border-slate-200 focus:border-blue-500'}`}
                                            >
                                                {CATEGORIES.map(cat => (
                                                    <option key={cat.id} value={cat.id}>{cat.icon} {cat.label}</option>
                                                ))}
                                            </select>
                                        </div>
                                    </div>
                                    <div className="space-y-1">
                                        <label className="text-xs font-semibold text-slate-500">지출 금액</label>
                                        <input
                                            type="number"
                                            placeholder="금액을 입력하세요"
                                            value={formData.amount}
                                            onChange={e => setFormData({ ...formData, amount: e.target.value })}
                                            className={`w-full p-3 rounded-xl outline-none border transition-all text-xl font-bold ${isDarkMode ? 'bg-slate-900 border-slate-700 focus:border-blue-500' : 'bg-slate-100 border-slate-200 focus:border-blue-500'}`}
                                        />
                                    </div>
                                    <div className="space-y-1">
                                        <label className="text-xs font-semibold text-slate-500">메모 (선택사항)</label>
                                        <input
                                            type="text"
                                            placeholder="지출 내용을 적어주세요"
                                            value={formData.memo}
                                            onChange={e => setFormData({ ...formData, memo: e.target.value })}
                                            className={`w-full p-3 rounded-xl outline-none border transition-all ${isDarkMode ? 'bg-slate-900 border-slate-700 focus:border-blue-500' : 'bg-slate-100 border-slate-200 focus:border-blue-500'}`}
                                        />
                                    </div>
                                    <button type="submit" className="w-full bg-indigo-600 font-bold py-3 rounded-xl hover:bg-indigo-500 active:scale-95 transition-all shadow-lg shadow-indigo-600/20">
                                        {formData.id ? '수정 완료' : '내역 추가'}
                                    </button>
                                    {formData.id && (
                                        <button
                                            type="button"
                                            onClick={() => setFormData({ id: null, category: 'food', amount: '', date: new Date().toISOString().split('T')[0], memo: '' })}
                                            className="w-full text-slate-500 text-sm font-medium py-2"
                                        >
                                            취소
                                        </button>
                                    )}
                                </form>
                            </div>
                        </div>

                        {/* Right: List */}
                        <div className="lg:col-span-7 space-y-4">
                            <div className="flex justify-between items-center mb-2 px-2">
                                <h3 className="text-xl font-bold">지출 내역</h3>
                                <select
                                    value={filterCategory}
                                    onChange={e => setFilterCategory(e.target.value)}
                                    className={`p-2 rounded-lg text-sm outline-none border ${isDarkMode ? 'bg-slate-800 border-slate-700' : 'bg-white border-slate-200'}`}
                                >
                                    <option value="all">모든 카테고리</option>
                                    {CATEGORIES.map(cat => (
                                        <option key={cat.id} value={cat.id}>{cat.label}</option>
                                    ))}
                                </select>
                            </div>

                            <div className={`space-y-3 max-h-[750px] overflow-y-auto pr-2 custom-scrollbar`}>
                                {filteredExpenses.length === 0 ? (
                                    <div className="py-20 text-center text-slate-500 italic">내역이 없습니다. 새로운 지출을 등록해보세요! ✨</div>
                                ) : (
                                    filteredExpenses.map(ex => {
                                        const category = CATEGORIES.find(c => c.id === ex.category);
                                        return (
                                            <div key={ex.id} className={`group p-4 rounded-2xl border transition-all border-white/5 animate-in ${isDarkMode ? 'bg-slate-800/40 hover:bg-slate-800' : 'bg-white shadow-sm hover:shadow-md'}`}>
                                                <div className="flex items-center gap-4">
                                                    <div className="w-12 h-12 rounded-2xl flex items-center justify-center text-2xl shadow-inner" style={{ backgroundColor: `${category?.color}20` }}>
                                                        {category?.icon}
                                                    </div>
                                                    <div className="flex-1">
                                                        <div className="flex justify-between items-start">
                                                            <div>
                                                                <p className="font-bold text-lg">{formatCurrency(ex.amount)}</p>
                                                                <p className="text-xs text-slate-500 font-medium">{category?.label} • {ex.date}</p>
                                                            </div>
                                                            <div className="flex gap-2 opacity-0 group-hover:opacity-100 transition-opacity">
                                                                <button onClick={() => handleEdit(ex)} className="p-2 hover:bg-blue-500/20 rounded-lg text-blue-400">✏️</button>
                                                                <button onClick={() => handleDelete(ex.id)} className="p-2 hover:bg-red-500/20 rounded-lg text-red-400">🗑️</button>
                                                            </div>
                                                        </div>
                                                        {ex.memo && <p className={`mt-2 text-sm italic py-1 px-3 rounded-lg ${isDarkMode ? 'bg-slate-900/50 text-slate-400' : 'bg-slate-50 text-slate-600'}`}>{ex.memo}</p>}
                                                    </div>
                                                </div>
                                            </div>
                                        );
                                    })
                                )}
                            </div>
                        </div>
                    </main>

                    <footer className="p-10 text-center text-slate-500 text-xs">
                        &copy; 2026 ExpenseTracker Pro • Made with ❤️ by Kodari
                    </footer>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<ExpenseApp />);
    </script>
</body>

</html>

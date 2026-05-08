<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>RIFA PROFESIONAL - MAYO 29</title>
    <script src="https://cdn.tailwindcss.com"></script>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getFirestore, doc, onSnapshot, setDoc } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";

        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);
        
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'rifa_produccion_final_mayo';

        let rifaData = Array(100).fill(null);
        let user = null;
        let currentNumIndex = null;
        let isAdminAuthenticated = false;
        const CLAVE_MAESTRA = "1234";

        const initAuth = async () => {
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Error de acceso:", error);
            }
        };

        onAuthStateChanged(auth, (u) => {
            user = u;
            if (user) conectarBaseDeDatos();
        });

        function getDocRef() {
            return doc(db, 'artifacts', appId, 'public', 'data', 'rifa_global', 'estado_oficial');
        }

        function conectarBaseDeDatos() {
            const docRef = getDocRef();
            onSnapshot(docRef, (docSnap) => {
                if (docSnap.exists()) {
                    const data = docSnap.data();
                    rifaData = data.numeros || Array(100).fill(null);
                } else {
                    actualizarServidor(Array(100).fill(null));
                }
                renderizarPantalla();
                if (!document.getElementById('panel-admin').classList.contains('hidden')) {
                    actualizarListaAdmin();
                }
            }, (error) => console.error("Error de conexión:", error));
        }

        async function actualizarServidor(nuevaData) {
            if (!user) return;
            try {
                const docRef = getDocRef();
                const dataLimpia = nuevaData.map(item => item ? { 
                    nombre: item.nombre, 
                    telefono: item.telefono,
                    metodo: item.metodo || null 
                } : null);
                await setDoc(docRef, { numeros: dataLimpia });
            } catch (error) { console.error("Error al guardar:", error); }
        }

        window.renderizarPantalla = function() {
            const grid = document.getElementById('grid-rifa');
            if (!grid) return;
            grid.innerHTML = '';
            let contador = 0;
            rifaData.forEach((dato, i) => {
                const ocupado = dato !== null;
                const celda = document.createElement('div');
                celda.className = `h-14 sm:h-16 flex items-center justify-center rounded-2xl text-2xl font-black cursor-pointer transition-all active:scale-90 select-none shadow-sm border-2 ${ocupado ? 'bg-gradient-to-br from-orange-500 to-orange-700 text-white border-orange-800' : 'bg-white text-slate-800 border-slate-100'}`;
                celda.innerText = i.toString().padStart(2, '0');
                celda.onclick = () => abrirVentana(i);
                grid.appendChild(celda);
                if (ocupado) contador++;
            });
            document.getElementById('vendidos-count').innerText = contador;
        };

        window.abrirVentana = function(index) {
            currentNumIndex = index;
            const data = rifaData[index];
            document.getElementById('num-badge').innerText = `#${index.toString().padStart(2, '0')}`;
            
            if (data) {
                document.getElementById('modal-info-nombre').innerText = data.nombre;
                document.getElementById('modal-info-tel').innerText = data.telefono || "Sin teléfono";
                document.getElementById('view-registro').classList.add('hidden');
                document.getElementById('view-ocupado').classList.remove('hidden');
                document.getElementById('btn-guardar-modal').classList.add('hidden');
                document.getElementById('btn-liberar-directo').classList.remove('hidden');
            } else {
                document.getElementById('in-nombre').value = '';
                document.getElementById('in-tel').value = '';
                document.getElementById('view-registro').classList.remove('hidden');
                document.getElementById('view-ocupado').classList.add('hidden');
                document.getElementById('btn-guardar-modal').classList.remove('hidden');
                document.getElementById('btn-liberar-directo').classList.add('hidden');
            }
            document.getElementById('modal-main').classList.remove('hidden');
        };

        window.cerrarVentana = function() {
            document.getElementById('modal-main').classList.add('hidden');
        };

        window.guardarReserva = async function() {
            const nom = document.getElementById('in-nombre').value.trim();
            const tel = document.getElementById('in-tel').value.trim();
            if (!nom) return;
            rifaData[currentNumIndex] = { nombre: nom, telefono: tel, metodo: null };
            await actualizarServidor(rifaData);
            cerrarVentana();
        };

        window.solicitarAdmin = function() {
            if (isAdminAuthenticated) {
                document.getElementById('panel-admin').classList.remove('hidden');
                actualizarListaAdmin();
            } else {
                document.getElementById('input-pass').value = '';
                document.getElementById('modal-pass').classList.remove('hidden');
                window.adminActionCallback = () => {
                    document.getElementById('panel-admin').classList.remove('hidden');
                    actualizarListaAdmin();
                };
            }
        };

        window.liberarConClave = function() {
            document.getElementById('input-pass').value = '';
            document.getElementById('modal-pass').classList.remove('hidden');
            window.adminActionCallback = async () => {
                rifaData[currentNumIndex] = null;
                await actualizarServidor(rifaData);
                cerrarVentana();
            };
        };

        window.verificarClave = async function() {
            if (document.getElementById('input-pass').value === CLAVE_MAESTRA) {
                isAdminAuthenticated = true;
                document.getElementById('modal-pass').classList.add('hidden');
                if (window.adminActionCallback) window.adminActionCallback();
            }
        };

        window.copiarTexto = function(texto, idAviso) {
            const el = document.createElement('textarea');
            el.value = texto.replace(/\s/g, '').replace(/-/g, '');
            document.body.appendChild(el);
            el.select();
            document.execCommand('copy');
            document.body.removeChild(el);
            
            const aviso = document.getElementById(idAviso);
            const original = aviso.innerText;
            aviso.innerText = "¡COPIADO!";
            aviso.classList.add('text-green-500');
            setTimeout(() => {
                aviso.innerText = original;
                aviso.classList.remove('text-green-500');
            }, 1500);
        };

        window.cambiarMetodo = async function(idx, metodo) {
            rifaData[idx].metodo = (rifaData[idx].metodo === metodo) ? null : metodo;
            await actualizarServidor(rifaData);
        };

        window.liberarDesdeAdmin = async function(idx) {
            rifaData[idx] = null;
            await actualizarServidor(rifaData);
        };

        function actualizarListaAdmin() {
            const listado = document.getElementById('lista-cuerpo');
            listado.innerHTML = '';
            rifaData.forEach((d, i) => {
                if (d) {
                    listado.innerHTML += `
                        <div class="p-4 bg-white border-b flex items-center justify-between">
                            <div class="flex items-center gap-3">
                                <span class="font-impact text-2xl text-orange-600 w-10">#${i.toString().padStart(2, '0')}</span>
                                <div class="overflow-hidden">
                                    <div class="font-bold uppercase text-[11px] text-slate-700 truncate w-32 leading-none">${d.nombre}</div>
                                    <div class="text-[10px] text-slate-400 font-mono">${d.telefono}</div>
                                </div>
                            </div>
                            <div class="flex items-center gap-1">
                                <button onclick="cambiarMetodo(${i}, 'Nequi')" class="text-[9px] font-black px-2 py-2 rounded border-2 transition-colors ${d.metodo === 'Nequi' ? 'bg-[#FF00BF] text-white border-[#FF00BF]' : 'bg-white text-slate-300 border-slate-50'}">NEQUI</button>
                                <button onclick="cambiarMetodo(${i}, 'Bancolombia')" class="text-[9px] font-black px-2 py-2 rounded border-2 transition-colors ${d.metodo === 'Bancolombia' ? 'bg-slate-900 text-white border-slate-900' : 'bg-white text-slate-300 border-slate-50'}">BC</button>
                                <button onclick="liberarDesdeAdmin(${i})" class="ml-2 p-2 text-red-400">
                                    <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M3 6h18"/><path d="M19 6v14c0 1-1 2-2 2H7c-1 0-2-1-2-2V6"/><path d="M8 6V4c0-1 1-2 2-2h4c1 0 2 1 2 2v2"/></svg>
                                </button>
                            </div>
                        </div>`;
                }
            });
        }

        initAuth();
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Bebas+Neue&family=Inter:wght@400;700;900&family=Oswald:wght@700&display=swap');
        .font-impact { font-family: 'Bebas Neue', cursive; }
        .font-main { font-family: 'Inter', sans-serif; }
        .font-prize { font-family: 'Oswald', sans-serif; }
        
        .bg-blur { background: rgba(15, 23, 42, 0.96); backdrop-filter: blur(10px); }
        .gradient-header { background: linear-gradient(180deg, #020617 0%, #0f172a 100%); }
        
        .title-stroke { 
            color: #f97316; 
            -webkit-text-stroke: 1px #ffffff; 
            paint-order: stroke fill; 
            filter: drop-shadow(2px 3px 0px rgba(0,0,0,0.8)); 
        }
        
        .prize-glow {
            color: #fff;
            background: linear-gradient(to bottom, #fff 30%, #fdba74 100%);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
            line-height: 1.1;
            padding: 0.2em 0.3em;
            display: inline-block;
            filter: drop-shadow(0 0 12px rgba(249, 115, 22, 0.4));
            font-style: italic;
            white-space: nowrap;
        }

        .valor-resaltado { 
            background: #f97316; 
            box-shadow: 0 5px 15px rgba(249, 115, 22, 0.4); 
            border: 1px solid rgba(255,255,255,0.3);
            padding: 12px 30px;
            display: inline-flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
        }
    </style>
</head>

<body class="bg-slate-100 min-h-screen font-main text-slate-900 select-none overflow-x-hidden">

    <!-- MODAL PRINCIPAL -->
    <div id="modal-main" class="bg-blur fixed inset-0 z-50 hidden flex items-center justify-center p-4">
        <div class="bg-white rounded-[2.5rem] w-full max-w-sm overflow-hidden shadow-2xl">
            <div class="p-8 text-center bg-slate-50 border-b">
                <p id="num-badge" class="text-9xl font-impact text-orange-600 leading-none">#--</p>
            </div>
            <div class="p-8 space-y-4">
                <div id="view-registro" class="space-y-4">
                    <input type="text" id="in-nombre" class="w-full bg-slate-50 border-2 border-slate-100 p-5 rounded-2xl font-bold outline-none focus:border-orange-500" placeholder="Nombre completo">
                    <input type="tel" id="in-tel" class="w-full bg-slate-50 border-2 border-slate-100 p-5 rounded-2xl font-bold outline-none focus:border-orange-500" placeholder="WhatsApp">
                </div>
                <div id="view-ocupado" class="hidden text-center space-y-4 py-4">
                    <div class="bg-orange-50 p-6 rounded-2xl">
                        <p class="text-[10px] font-black text-orange-400 uppercase tracking-widest mb-1">REGISTRADO POR</p>
                        <p id="modal-info-nombre" class="text-2xl font-bold text-slate-800 leading-tight">---</p>
                        <p id="modal-info-tel" class="text-sm font-mono text-slate-400 mt-1">---</p>
                    </div>
                </div>
                <button id="btn-guardar-modal" onclick="guardarReserva()" class="w-full bg-orange-600 text-white py-5 rounded-2xl font-impact text-3xl uppercase tracking-widest shadow-lg active:scale-95 transition-all">RESERVAR AHORA</button>
                <button id="btn-liberar-directo" onclick="liberarConClave()" class="hidden w-full bg-red-600 text-white py-4 rounded-2xl font-black text-lg uppercase tracking-widest active:scale-95 transition-all shadow-md mt-2">LIBERAR CUPO</button>
                <button onclick="cerrarVentana()" class="w-full text-slate-400 font-black text-[10px] uppercase py-2 tracking-widest">Cerrar</button>
            </div>
        </div>
    </div>

    <!-- MODAL PASS -->
    <div id="modal-pass" class="bg-blur fixed inset-0 z-[60] hidden flex items-center justify-center p-4">
        <div class="bg-white rounded-[2.5rem] p-10 w-full max-w-xs text-center shadow-2xl">
            <h3 class="font-impact text-3xl uppercase mb-6 tracking-wide text-slate-800">Seguridad</h3>
            <input type="password" id="input-pass" class="w-full bg-slate-50 border-2 p-5 rounded-2xl text-center font-bold mb-6 text-4xl outline-none focus:border-orange-500" placeholder="••••" inputmode="numeric">
            <button onclick="verificarClave()" class="w-full bg-slate-900 text-white py-5 rounded-2xl font-bold uppercase shadow-lg active:scale-95">Confirmar</button>
            <button onclick="document.getElementById('modal-pass').classList.add('hidden')" class="mt-4 text-[10px] font-black text-slate-300 uppercase tracking-widest">Cancelar</button>
        </div>
    </div>

    <!-- PANEL ADMIN -->
    <div id="panel-admin" class="hidden fixed inset-0 z-[70] bg-slate-50 flex flex-col">
        <div class="bg-white p-6 border-b flex justify-between items-center shadow-sm">
            <div>
                <h2 class="text-2xl font-impact uppercase italic text-slate-800">Control Pagos</h2>
                <p class="text-[9px] font-bold text-slate-400 uppercase tracking-widest">Gestión de recaudos</p>
            </div>
            <button onclick="document.getElementById('panel-admin').classList.add('hidden')" class="w-12 h-12 flex items-center justify-center bg-slate-100 rounded-full text-3xl font-light">×</button>
        </div>
        <div id="lista-cuerpo" class="flex-1 overflow-y-auto bg-white pb-10"></div>
    </div>

    <div class="max-w-xl mx-auto bg-white min-h-screen shadow-2xl overflow-hidden relative pb-10">
        
        <div class="gradient-header px-4 py-12 text-white text-center">
            <h1 class="font-impact text-7xl sm:text-8xl uppercase title-stroke italic mb-4">GRAN RIFA</h1>
            
            <div class="mb-2">
                <div class="inline-block bg-orange-600/30 border border-orange-500/50 px-5 py-1.5 rounded-full shadow-lg">
                    <span class="text-xs sm:text-base font-black uppercase tracking-widest text-orange-300">Mayo 29 - Lotería de Medellín</span>
                </div>
            </div>
            
            <div class="mb-2 flex justify-center items-center">
                <p class="font-prize text-[4.5rem] sm:text-[6.5rem] prize-glow uppercase">
                    $1.000.000
                </p>
            </div>

            <div class="flex justify-center mt-2">
                <div class="valor-resaltado rounded-2xl transform -rotate-1 border-2 border-white/20">
                    <span class="text-[10px] font-black uppercase tracking-widest text-white/80 block mb-1">VALOR</span>
                    <span class="font-impact text-4xl leading-none text-white">$20.000</span>
                </div>
            </div>
        </div>

        <div class="px-2 py-5 grid grid-cols-3 gap-2 bg-slate-50/80 border-b">
            <div onclick="copiarTexto('000-0000-00', 'bc-label')" class="bg-white p-3 rounded-2xl border border-slate-100 flex flex-col items-center justify-center text-center cursor-pointer active:scale-95 transition-transform shadow-sm">
                <span id="bc-label" class="font-black text-slate-400 text-[8px] uppercase tracking-tighter mb-1">BANCOLOMBIA</span>
                <p class="font-impact text-sm text-slate-700 tracking-wider">000-0000-00</p>
            </div>
            
            <div onclick="copiarTexto('3000000000', 'nq-label')" class="bg-white p-3 rounded-2xl border border-slate-100 flex flex-col items-center justify-center text-center border-t-2 border-t-[#FF00BF] cursor-pointer active:scale-95 transition-transform shadow-sm">
                <span id="nq-label" class="font-black text-[#FF00BF] text-[8px] uppercase tracking-tighter mb-1">NEQUI</span>
                <p class="font-impact text-sm text-slate-700 tracking-wider">300 000 0000</p>
            </div>

            <div onclick="solicitarAdmin()" class="bg-white p-3 rounded-2xl border border-slate-100 flex flex-col items-center justify-center text-center cursor-pointer active:bg-orange-50 shadow-sm relative overflow-hidden">
                <span id="vendidos-count" class="font-impact text-2xl text-orange-600 leading-none">0</span>
                <span class="text-[8px] font-bold text-slate-300 uppercase leading-none tracking-tighter">VENDIDOS</span>
            </div>
        </div>

        <div id="grid-rifa" class="grid grid-cols-4 sm:grid-cols-5 gap-3 p-6 pb-20"></div>
        
        <div class="text-center py-8 opacity-20">
            <p class="text-[7px] font-black text-slate-400 uppercase tracking-[1em]">PLATAFORMA PREMIUM v3.5</p>
        </div>
    </div>
</body>
</html>

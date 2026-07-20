import React, { useState, useEffect, useMemo, useCallback, useRef } from "react";
import {
  ChevronRight, ChevronDown, Plus, Trash2, StickyNote,
  Search, Download, Upload, X, Check, RotateCcw
} from "lucide-react";

/* ---------- Configuração de status ---------- */
const STATUS = {
  nao_mapeado:  { nome: "Não Mapeado",  cor: "#ff9a8f" }, // vermelho claro
  mapeado:      { nome: "Mapeado",      cor: "#7fb8ff" }, // azul claro
  em_progresso: { nome: "Em Progresso", cor: "#ffd572" }, // amarelo claro
  concluido:    { nome: "Concluído",    cor: "#8fdca0" }, // verde claro
};
const ORDEM = ["nao_mapeado", "mapeado", "em_progresso", "concluido"];

const uid = () =>
  (typeof crypto !== "undefined" && crypto.randomUUID)
    ? crypto.randomUUID()
    : Math.random().toString(36).slice(2) + Date.now().toString(36);

const novoNo = (titulo, status = "nao_mapeado") => ({
  id: uid(), titulo, status, anotacoes: "",
  aberto: true, notasAbertas: false, filhos: [],
});

const dadosIniciais = () => ([
  {
    id: uid(), titulo: "Tema 01", status: "em_progresso",
    aberto: true, notasAbertas: false, anotacoes: "", filhos: [
      { id: uid(), titulo: "exemplo 01", status: "concluido", aberto: true, notasAbertas: false, anotacoes: "", filhos: [] },
      {
        id: uid(), titulo: "exemplo 02", status: "em_progresso",
        aberto: true, notasAbertas: true,
        anotacoes: "Anotações livres deste subconjunto — descreva aqui os pontos que ainda faltam mapear e o que já foi concluído.",
        filhos: [
          { id: uid(), titulo: "Ponto ainda não analisado", status: "nao_mapeado", aberto: true, notasAbertas: false, anotacoes: "", filhos: [] },
          { id: uid(), titulo: "Ponto já identificado", status: "mapeado", aberto: true, notasAbertas: false, anotacoes: "", filhos: [] },
        ],
      },
    ],
  },
]);

/* ---------- Helpers imutáveis sobre a árvore ---------- */
const atualizar = (nodes, id, patch) =>
  nodes.map((n) =>
    n.id === id
      ? { ...n, ...(typeof patch === "function" ? patch(n) : patch) }
      : { ...n, filhos: atualizar(n.filhos, id, patch) }
  );

const inserirFilho = (nodes, paiId, novo) =>
  nodes.map((n) =>
    n.id === paiId
      ? { ...n, aberto: true, filhos: [...n.filhos, novo] }
      : { ...n, filhos: inserirFilho(n.filhos, paiId, novo) }
  );

const remover = (nodes, id) =>
  nodes.filter((n) => n.id !== id).map((n) => ({ ...n, filhos: remover(n.filhos, id) }));

const contar = (nodes, acc) => {
  nodes.forEach((n) => {
    acc.total++;
    acc[n.status] = (acc[n.status] || 0) + 1;
    contar(n.filhos, acc);
  });
  return acc;
};

const progresso = (node) => {
  let tot = 0, conc = 0;
  const walk = (ns) => ns.forEach((n) => { tot++; if (n.status === "concluido") conc++; walk(n.filhos); });
  walk(node.filhos);
  if (tot === 0) return node.status === "concluido" ? 100 : 0;
  return Math.round((conc / tot) * 100);
};

const filtrar = (nodes, texto, statusSel) => {
  const t = texto.trim().toLowerCase();
  const res = [];
  for (const n of nodes) {
    const filhosF = filtrar(n.filhos, texto, statusSel);
    const okTexto = !t || n.titulo.toLowerCase().includes(t) || (n.anotacoes || "").toLowerCase().includes(t);
    const okStatus = !statusSel || n.status === statusSel;
    const selfMatch = okTexto && okStatus;
    if (selfMatch || filhosF.length) {
      res.push({ ...n, filhos: filhosF, _atenuado: !selfMatch, aberto: (t || statusSel) ? true : n.aberto });
    }
  }
  return res;
};

/* ---------- Nó recursivo (definido no escopo do módulo p/ manter foco em inputs) ---------- */
function No({ node, depth, ctx }) {
  const st = STATUS[node.status];
  const temFilhos = node.filhos.length > 0;
  const escala = Math.max(0.82, 1 - depth * 0.045);
  const editando = ctx.editandoId === node.id;
  const menuAberto = ctx.menuId === node.id;
  const confirmando = ctx.confirmId === node.id;
  const prog = depth === 0 ? progresso(node) : null;

  return (
    <div className="no-wrap" style={{ "--cor-no": st.cor }}>
      <div
        className={"card" + (depth === 0 ? " card-tema" : "") + (node._atenuado ? " atenuado" : "")}
        style={{ fontSize: `${escala}rem` }}
      >
        {/* expandir / recolher */}
        {temFilhos ? (
          <button className="ico expand" title={node.aberto ? "Recolher" : "Expandir"} onClick={() => ctx.toggleAberto(node.id)}>
            {node.aberto ? <ChevronDown size={16} /> : <ChevronRight size={16} />}
          </button>
        ) : (
          <span className="expand-vazio" />
        )}

        {/* bolinha de status -> abre menu */}
        <button
          className="dot"
          style={{ background: st.cor, boxShadow: `0 0 0 3px ${st.cor}22` }}
          title={st.nome}
          onClick={() => ctx.setMenuId(menuAberto ? null : node.id)}
        />

        {/* título (duplo clique edita) */}
        {editando ? (
          <input
            className="titulo-input"
            autoFocus
            value={ctx.rascunho}
            onChange={(e) => ctx.setRascunho(e.target.value)}
            onBlur={() => ctx.commitTitulo(node.id)}
            onKeyDown={(e) => {
              if (e.key === "Enter") ctx.commitTitulo(node.id);
              if (e.key === "Escape") ctx.setEditandoId(null);
            }}
          />
        ) : (
          <span
            className="titulo"
            title="Duplo clique para renomear"
            onDoubleClick={() => ctx.iniciarEdicao(node)}
          >
            {node.titulo || "Sem título"}
          </span>
        )}

        {/* progresso do tema principal */}
        {depth === 0 && (
          <div className="prog">
            <div className="prog-trilha"><div className="prog-fill" style={{ width: `${prog}%` }} /></div>
            <span className="prog-num">{prog}%</span>
          </div>
        )}

        {/* ações */}
        <div className="acoes">
          <button
            className={"ico" + (node.notasAbertas ? " ativo" : "") + (node.anotacoes ? " tem-nota" : "")}
            title="Anotações livres"
            onClick={() => ctx.toggleNotas(node.id)}
          ><StickyNote size={15} /></button>
          <button className="ico" title="Adicionar ponto" onClick={() => ctx.addFilho(node.id)}><Plus size={16} /></button>
          <button className="ico perigo" title="Excluir" onClick={() => ctx.setConfirmId(confirmando ? null : node.id)}><Trash2 size={15} /></button>
        </div>
      </div>

      {/* menu de status */}
      {menuAberto && (
        <div className="menu-status" onMouseLeave={() => ctx.setMenuId(null)}>
          {ORDEM.map((k) => (
            <button key={k} className="opt" onClick={() => ctx.setStatus(node.id, k)}>
              <span className="opt-dot" style={{ background: STATUS[k].cor }} />
              <span>{STATUS[k].nome}</span>
              {node.status === k && <Check size={14} className="opt-check" />}
            </button>
          ))}
        </div>
      )}

      {/* confirmação de exclusão */}
      {confirmando && (
        <div className="confirmar">
          <span>Excluir "{node.titulo}"{temFilhos ? " e tudo dentro dele" : ""}?</span>
          <div className="confirmar-btns">
            <button className="btn-sim" onClick={() => ctx.remover(node.id)}>Excluir</button>
            <button className="btn-nao" onClick={() => ctx.setConfirmId(null)}>Cancelar</button>
          </div>
        </div>
      )}

      {/* anotações */}
      {node.notasAbertas && (
        <div className="notas">
          <textarea
            placeholder="Anotações livres deste ponto…"
            value={node.anotacoes}
            onChange={(e) => ctx.setNotas(node.id, e.target.value)}
          />
        </div>
      )}

      {/* filhos (ramos) */}
      {node.aberto && temFilhos && (
        <div className="ramos">
          {node.filhos.map((f) => <No key={f.id} node={f} depth={depth + 1} ctx={ctx} />)}
        </div>
      )}
    </div>
  );
}

/* ---------- App ---------- */
export default function App() {
  const [temas, setTemas] = useState([]);
  const [carregado, setCarregado] = useState(false);
  const [busca, setBusca] = useState("");
  const [filtroStatus, setFiltroStatus] = useState(null);

  // estado transitório de UI
  const [editandoId, setEditandoId] = useState(null);
  const [rascunho, setRascunho] = useState("");
  const [menuId, setMenuId] = useState(null);
  const [confirmId, setConfirmId] = useState(null);
  const fileRef = useRef(null);

  const temStorage = typeof window !== "undefined" && window.storage;

  /* carregar */
  useEffect(() => {
    (async () => {
      if (temStorage) {
        try {
          const r = await window.storage.get("mapa_mental_dados");
          if (r && r.value) { setTemas(JSON.parse(r.value)); setCarregado(true); return; }
        } catch (e) { /* chave inexistente */ }
      }
      setTemas(dadosIniciais());
      setCarregado(true);
    })();
  }, []); // eslint-disable-line

  /* salvar (debounce) */
  useEffect(() => {
    if (!carregado) return;
    const id = setTimeout(async () => {
      if (temStorage) {
        try { await window.storage.set("mapa_mental_dados", JSON.stringify(temas)); } catch (e) { /* ignora */ }
      }
    }, 400);
    return () => clearTimeout(id);
  }, [temas, carregado]); // eslint-disable-line

  /* ações */
  const commitTitulo = useCallback((id) => {
    setTemas((prev) => atualizar(prev, id, { titulo: rascunho.trim() || "Sem título" }));
    setEditandoId(null);
  }, [rascunho]);

  const iniciarEdicao = useCallback((node) => { setRascunho(node.titulo); setEditandoId(node.id); }, []);
  const toggleAberto = useCallback((id) => setTemas((p) => atualizar(p, id, (n) => ({ aberto: !n.aberto }))), []);
  const toggleNotas = useCallback((id) => setTemas((p) => atualizar(p, id, (n) => ({ notasAbertas: !n.notasAbertas }))), []);
  const setNotas = useCallback((id, texto) => setTemas((p) => atualizar(p, id, { anotacoes: texto })), []);
  const setStatus = useCallback((id, s) => { setTemas((p) => atualizar(p, id, { status: s })); setMenuId(null); }, []);
  const addFilho = useCallback((paiId) => setTemas((p) => inserirFilho(p, paiId, novoNo("Novo ponto"))), []);
  const removerNo = useCallback((id) => { setTemas((p) => remover(p, id)); setConfirmId(null); }, []);

  const ctx = useMemo(() => ({
    editandoId, setEditandoId, rascunho, setRascunho, menuId, setMenuId, confirmId, setConfirmId,
    commitTitulo, iniciarEdicao, toggleAberto, toggleNotas, setNotas, setStatus, addFilho, remover: removerNo,
  }), [editandoId, rascunho, menuId, confirmId, commitTitulo, iniciarEdicao, toggleAberto, toggleNotas, setNotas, setStatus, addFilho, removerNo]);

  /* derivados */
  const stats = useMemo(() => contar(temas, { total: 0, nao_mapeado: 0, mapeado: 0, em_progresso: 0, concluido: 0 }), [temas]);
  const progGeral = stats.total ? Math.round((stats.concluido / stats.total) * 100) : 0;
  const arvore = useMemo(
    () => (busca.trim() || filtroStatus) ? filtrar(temas, busca, filtroStatus) : temas,
    [temas, busca, filtroStatus]
  );

  const addTema = () => setTemas((p) => [...p, { ...novoNo("Novo tema", "nao_mapeado") }]);

  const exportar = () => {
    const blob = new Blob([JSON.stringify(temas, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url; a.download = "mapa-mental.json"; a.click();
    URL.revokeObjectURL(url);
  };
  const importar = (e) => {
    const f = e.target.files?.[0];
    if (!f) return;
    const reader = new FileReader();
    reader.onload = () => { try { const d = JSON.parse(reader.result); if (Array.isArray(d)) setTemas(d); } catch (err) {} };
    reader.readAsText(f);
    e.target.value = "";
  };
  const resetar = () => setTemas(dadosIniciais());

  return (
    <div className="raiz">
      <style>{css}</style>

      <div className="layout">
        {/* Painel lateral */}
        <aside className="lateral">
          <header className="marca">
            <div className="marca-glifo" aria-hidden>
              <span className="glifo-no glifo-centro" />
              <span className="glifo-linha l1" /><span className="glifo-linha l2" /><span className="glifo-linha l3" />
              <span className="glifo-no glifo-a" /><span className="glifo-no glifo-b" /><span className="glifo-no glifo-c" />
            </div>
            <div>
              <h1>Mapa Mental</h1>
              <p>Mapeie, acompanhe e conclua — um passo de cada vez.</p>
            </div>
          </header>

          <div className="prog-geral">
            <div className="prog-geral-topo">
              <span>Progresso geral</span><strong>{progGeral}%</strong>
            </div>
            <div className="prog-trilha grande"><div className="prog-fill" style={{ width: `${progGeral}%` }} /></div>
            <span className="prog-geral-sub">{stats.concluido} de {stats.total} pontos concluídos</span>
          </div>

          <div className="chips">
            {ORDEM.map((k) => {
              const ativo = filtroStatus === k;
              return (
                <button
                  key={k}
                  className={"chip" + (ativo ? " chip-ativo" : "")}
                  style={ativo ? { borderColor: STATUS[k].cor, background: `${STATUS[k].cor}1e` } : undefined}
                  onClick={() => setFiltroStatus(ativo ? null : k)}
                  title={ativo ? "Remover filtro" : "Filtrar por " + STATUS[k].nome}
                >
                  <span className="chip-dot" style={{ background: STATUS[k].cor }} />
                  <span className="chip-nome">{STATUS[k].nome}</span>
                  <span className="chip-num">{stats[k] || 0}</span>
                </button>
              );
            })}
          </div>

          <div className="busca">
            <Search size={15} />
            <input placeholder="Buscar em títulos e notas…" value={busca} onChange={(e) => setBusca(e.target.value)} />
            {busca && <button className="limpa" onClick={() => setBusca("")}><X size={14} /></button>}
          </div>

          <button className="btn-primario" onClick={addTema}><Plus size={16} /> Novo tema principal</button>

          <div className="ferramentas">
            <button className="btn-ghost" onClick={exportar}><Download size={14} /> Exportar</button>
            <button className="btn-ghost" onClick={() => fileRef.current?.click()}><Upload size={14} /> Importar</button>
            <button className="btn-ghost" onClick={resetar}><RotateCcw size={14} /> Reiniciar</button>
            <input ref={fileRef} type="file" accept="application/json" hidden onChange={importar} />
          </div>

          <p className="dica">
            Dica: duplo clique renomeia · a bolinha muda o status · <Plus size={11} /> cria um subponto dentro de qualquer nível.
          </p>
        </aside>

        {/* Tela do mapa */}
        <main className="tela">
          {(filtroStatus || busca) && (
            <div className="aviso-filtro">
              Exibindo apenas o que corresponde ao filtro.
              <button onClick={() => { setFiltroStatus(null); setBusca(""); }}>Limpar</button>
            </div>
          )}

          {arvore.length === 0 ? (
            <div className="vazio">
              {(busca || filtroStatus)
                ? <><p>Nenhum ponto corresponde ao filtro.</p><button className="btn-primario" onClick={() => { setFiltroStatus(null); setBusca(""); }}>Limpar filtros</button></>
                : <><p>Seu mapa está vazio.</p><button className="btn-primario" onClick={addTema}><Plus size={16} /> Criar o primeiro tema</button></>}
            </div>
          ) : (
            <div className="floresta">
              {arvore.map((t) => <No key={t.id} node={t} depth={0} ctx={ctx} />)}
              {!(busca || filtroStatus) && (
                <button className="add-tema-fim" onClick={addTema}><Plus size={16} /> Adicionar tema principal</button>
              )}
            </div>
          )}
        </main>
      </div>
    </div>
  );
}

/* ---------- Estilos ---------- */
const css = `
@import url('https://fonts.googleapis.com/css2?family=Ubuntu:wght@300;400;500;700&family=Ubuntu+Mono:wght@400;500&display=swap');

.raiz{
  --preto:#000000; --superficie:#1b1b1e; --superficie-2:#242428; --superficie-3:#2c2c31;
  --borda:#33333a; --texto:#e9eaee; --texto-dim:#9a9ca5; --texto-fraco:#6d6f79;
  --azul:#4c9bff; --azul-forte:#6cb0ff; --amarelo:#ffd572; --linha:#3a3a42;
  font-family:'Ubuntu',system-ui,sans-serif; color:var(--texto);
  background:var(--preto); min-height:100vh; -webkit-font-smoothing:antialiased;
}
.raiz *{box-sizing:border-box;}
.layout{display:grid; grid-template-columns:300px 1fr; min-height:100vh;}

/* ---- lateral ---- */
.lateral{
  background:linear-gradient(180deg,#0d0d0f 0%,#000 100%);
  border-right:1px solid var(--borda); padding:22px 18px; display:flex; flex-direction:column; gap:18px;
  position:sticky; top:0; height:100vh; overflow-y:auto;
}
.marca{display:flex; gap:12px; align-items:center;}
.marca h1{font-size:19px; font-weight:700; letter-spacing:-.02em; margin:0;}
.marca p{font-size:11.5px; color:var(--texto-dim); margin:3px 0 0; line-height:1.35;}
.marca-glifo{position:relative; width:40px; height:40px; flex:0 0 40px;}
.glifo-no{position:absolute; border-radius:50%; background:var(--azul);}
.glifo-centro{width:11px;height:11px; left:15px; top:15px; box-shadow:0 0 0 4px #4c9bff33;}
.glifo-a{width:7px;height:7px; left:1px; top:3px; background:var(--azul-forte);}
.glifo-b{width:7px;height:7px; right:0px; top:14px; background:var(--amarelo);}
.glifo-c{width:7px;height:7px; left:6px; bottom:1px; background:var(--azul-forte);}
.glifo-linha{position:absolute; height:1.5px; background:var(--linha); transform-origin:left center;}
.l1{width:16px; left:16px; top:19px; transform:rotate(-28deg);}
.l2{width:15px; left:19px; top:20px; transform:rotate(18deg);}
.l3{width:15px; left:15px; top:22px; transform:rotate(150deg);}

.prog-geral{background:var(--superficie); border:1px solid var(--borda); border-radius:14px; padding:13px 14px;}
.prog-geral-topo{display:flex; justify-content:space-between; align-items:baseline; font-size:12.5px; color:var(--texto-dim);}
.prog-geral-topo strong{color:var(--texto); font-size:15px;}
.prog-geral-sub{font-size:11px; color:var(--texto-fraco); margin-top:7px; display:block;}
.prog-trilha{height:6px; background:#2a2a30; border-radius:99px; overflow:hidden; margin-top:8px;}
.prog-trilha.grande{height:8px;}
.prog-fill{height:100%; background:linear-gradient(90deg,var(--azul),var(--azul-forte)); border-radius:99px; transition:width .35s ease;}

.chips{display:flex; flex-direction:column; gap:7px;}
.chip{display:flex; align-items:center; gap:9px; width:100%; padding:9px 11px; border-radius:11px;
  background:var(--superficie); border:1px solid var(--borda); color:var(--texto); cursor:pointer;
  font-family:inherit; font-size:13px; transition:background .15s,border-color .15s;}
.chip:hover{background:var(--superficie-2);}
.chip-ativo{background:var(--superficie-2);}
.chip-dot{width:9px;height:9px;border-radius:50%; flex:0 0 auto;}
.chip-nome{flex:1; text-align:left;}
.chip-num{font-family:'Ubuntu Mono',monospace; color:var(--texto-dim); font-size:12.5px;}

.busca{display:flex; align-items:center; gap:8px; background:var(--superficie); border:1px solid var(--borda);
  border-radius:11px; padding:0 10px; color:var(--texto-dim);}
.busca input{flex:1; background:none; border:none; outline:none; color:var(--texto); font-family:inherit;
  font-size:13px; padding:10px 0;}
.busca input::placeholder{color:var(--texto-fraco);}
.busca .limpa{background:none;border:none;color:var(--texto-dim);cursor:pointer;display:flex;padding:2px;}
.busca:focus-within{border-color:var(--azul);}

.btn-primario{display:flex; align-items:center; justify-content:center; gap:8px; padding:11px; border-radius:12px;
  background:var(--azul); color:#04121f; font-weight:600; font-family:inherit; font-size:13.5px; border:none;
  cursor:pointer; transition:filter .15s,transform .05s;}
.btn-primario:hover{filter:brightness(1.08);}
.btn-primario:active{transform:translateY(1px);}

.ferramentas{display:grid; grid-template-columns:1fr 1fr; gap:7px;}
.btn-ghost{display:flex; align-items:center; justify-content:center; gap:6px; padding:9px; border-radius:10px;
  background:transparent; border:1px solid var(--borda); color:var(--texto-dim); font-family:inherit; font-size:12px;
  cursor:pointer; transition:border-color .15s,color .15s;}
.btn-ghost:hover{border-color:var(--azul); color:var(--texto);}
.ferramentas .btn-ghost:last-of-type{grid-column:1 / -1;}

.dica{font-size:11px; color:var(--texto-fraco); line-height:1.55; margin:2px 0 0;}
.dica svg{display:inline; vertical-align:-1px;}

/* ---- tela ---- */
.tela{padding:26px 30px 90px; overflow-x:auto;}
.aviso-filtro{display:flex; align-items:center; gap:12px; font-size:12.5px; color:var(--texto-dim); margin-bottom:16px;}
.aviso-filtro button{background:none; border:1px solid var(--borda); color:var(--azul); border-radius:8px;
  padding:4px 10px; font-family:inherit; font-size:12px; cursor:pointer;}
.floresta{display:flex; flex-direction:column; gap:14px; max-width:900px;}

.vazio{display:flex; flex-direction:column; align-items:center; gap:16px; padding:80px 20px; text-align:center;}
.vazio p{color:var(--texto-dim); font-size:14px;}

/* ---- nós / ramos ---- */
.no-wrap{position:relative;}
.ramos{margin-left:16px; padding-top:2px; position:relative;}
.ramos .no-wrap{position:relative; padding-left:26px; margin-top:8px;}
.ramos .no-wrap::before{content:''; position:absolute; left:0; top:0; width:2px; height:100%; background:var(--linha);}
.ramos .no-wrap:last-child::before{height:23px;}
.ramos .no-wrap::after{content:''; position:absolute; left:0; top:22px; width:20px; height:2px;
  background:var(--cor-no); opacity:.55; border-radius:2px;}

.card{display:flex; align-items:center; gap:9px; background:var(--superficie); border:1px solid var(--borda);
  border-radius:14px; padding:9px 11px; transition:border-color .15s,background .15s,opacity .2s;
  position:relative; z-index:1;}
.card:hover{border-color:#42424c;}
.card-tema{background:var(--superficie-2); border-color:#3c3c45; padding:12px 14px;}
.card.atenuado{opacity:.4;}

.ico{display:flex; align-items:center; justify-content:center; width:28px; height:28px; border-radius:8px;
  background:transparent; border:none; color:var(--texto-dim); cursor:pointer; transition:background .15s,color .15s; flex:0 0 auto;}
.ico:hover{background:var(--superficie-3); color:var(--texto);}
.ico.ativo{color:var(--azul); background:#4c9bff1c;}
.ico.tem-nota{color:var(--amarelo);}
.ico.perigo:hover{background:#ff6b5622; color:#ff9a8f;}
.expand{color:var(--texto-dim);}
.expand-vazio{width:28px; flex:0 0 auto;}

.dot{width:15px; height:15px; border-radius:50%; border:none; cursor:pointer; flex:0 0 auto;
  transition:transform .12s;}
.dot:hover{transform:scale(1.15);}

.titulo{flex:1; font-size:1em; color:var(--texto); cursor:text; padding:4px 2px; line-height:1.3; word-break:break-word;}
.card-tema .titulo{font-weight:600; letter-spacing:-.01em;}
.titulo-input{flex:1; font-family:inherit; font-size:1em; background:#101013; border:1px solid var(--azul);
  color:var(--texto); border-radius:8px; padding:5px 8px; outline:none;}

.prog{display:flex; align-items:center; gap:8px; flex:0 0 auto; margin-left:auto;}
.prog .prog-trilha{width:84px; margin:0;}
.prog-num{font-family:'Ubuntu Mono',monospace; font-size:12px; color:var(--texto-dim); min-width:34px; text-align:right;}

.acoes{display:flex; align-items:center; gap:2px; flex:0 0 auto;}
.card-tema .acoes{margin-left:10px;}

/* menu status */
.menu-status{position:absolute; z-index:20; left:44px; margin-top:4px; background:var(--superficie-2);
  border:1px solid var(--borda); border-radius:12px; padding:5px; box-shadow:0 12px 30px #000a; min-width:172px;}
.opt{display:flex; align-items:center; gap:9px; width:100%; padding:8px 10px; border-radius:8px; background:none;
  border:none; color:var(--texto); font-family:inherit; font-size:13px; cursor:pointer; text-align:left;}
.opt:hover{background:var(--superficie-3);}
.opt-dot{width:10px;height:10px;border-radius:50%;}
.opt-check{margin-left:auto; color:var(--azul);}

/* confirmar */
.confirmar{display:flex; align-items:center; justify-content:space-between; gap:12px; flex-wrap:wrap;
  margin-top:6px; margin-left:2px; background:#1f1416; border:1px solid #52303099; border-radius:11px;
  padding:9px 12px; font-size:12.5px; color:#ffb3ab;}
.confirmar-btns{display:flex; gap:7px;}
.btn-sim{background:#ff6b56; color:#1a0705; border:none; border-radius:8px; padding:6px 12px; font-family:inherit;
  font-weight:600; font-size:12px; cursor:pointer;}
.btn-nao{background:transparent; color:var(--texto-dim); border:1px solid var(--borda); border-radius:8px;
  padding:6px 12px; font-family:inherit; font-size:12px; cursor:pointer;}

/* notas */
.notas{margin:7px 0 2px 2px;}
.notas textarea{width:100%; min-height:66px; resize:vertical; background:#101013; border:1px solid var(--borda);
  border-radius:11px; padding:11px 12px; color:var(--texto); font-family:inherit; font-size:13px; line-height:1.5; outline:none;}
.notas textarea:focus{border-color:var(--azul);}
.notas textarea::placeholder{color:var(--texto-fraco);}

.add-tema-fim{display:flex; align-items:center; gap:8px; align-self:flex-start; margin-top:6px; padding:10px 16px;
  border-radius:12px; background:transparent; border:1px dashed var(--borda); color:var(--texto-dim);
  font-family:inherit; font-size:13px; cursor:pointer; transition:border-color .15s,color .15s;}
.add-tema-fim:hover{border-color:var(--azul); color:var(--texto);}

.raiz{animation:aparecer .4s ease;}
@keyframes aparecer{from{opacity:0} to{opacity:1}}

/* responsivo */
@media (max-width:760px){
  .layout{grid-template-columns:1fr;}
  .lateral{position:static; height:auto; border-right:none; border-bottom:1px solid var(--borda);}
  .tela{padding:18px 14px 80px;}
  .prog .prog-trilha{width:54px;}
}
@media (prefers-reduced-motion:reduce){
  *{animation:none !important; transition:none !important;}
}
`;

import React, { useState, useEffect, useMemo, useCallback } from "react";
import { Trash2, Plus } from "lucide-react";

const uid = () => `g_${Date.now().toString(36)}${Math.random().toString(36).slice(2, 7)}`;

const money = (n) =>
  (n < 0 ? "-" : "") +
  "R$ " +
  Math.abs(n).toLocaleString("pt-BR", { minimumFractionDigits: 2, maximumFractionDigits: 2 });

function useStoredValue(key, fallback) {
  const [data, setData] = useState(fallback);
  const [loaded, setLoaded] = useState(false);

  useEffect(() => {
    let cancelled = false;
    (async () => {
      try {
        const res = await window.storage.get(key, false);
        if (!cancelled) {
          setData(res && res.value ? JSON.parse(res.value) : fallback);
          setLoaded(true);
        }
      } catch (e) {
        if (!cancelled) {
          setData(fallback);
          setLoaded(true);
        }
      }
    })();
    return () => {
      cancelled = true;
    };
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [key]);

  const persist = useCallback(
    async (next) => {
      setData(next);
      try {
        await window.storage.set(key, JSON.stringify(next), false);
      } catch (e) {
        // silent fail, data still reflected in this session
      }
    },
    [key]
  );

  return [data, persist, loaded];
}

export default function App() {
  const [gastos, setGastos, gastosLoaded] = useStoredValue("simples:gastos", []);
  const [entrada, setEntrada, entradaLoaded] = useStoredValue("simples:entrada", "");

  const [nome, setNome] = useState("");
  const [valor, setValor] = useState("");

  const ready = gastosLoaded && entradaLoaded;

  const totalGastos = useMemo(() => gastos.reduce((s, g) => s + g.valor, 0), [gastos]);
  const entradaNum = Number(entrada) || 0;
  const saldo = entradaNum - totalGastos;

  const canAdd = nome.trim() && Number(valor) > 0;

  const addGasto = () => {
    if (!canAdd) return;
    setGastos([{ id: uid(), nome: nome.trim(), valor: Number(valor) }, ...gastos]);
    setNome("");
    setValor("");
  };

  const removeGasto = (id) => {
    setGastos(gastos.filter((g) => g.id !== id));
  };

  const updateGastoNome = (id, novoNome) => {
    setGastos(gastos.map((g) => (g.id === id ? { ...g, nome: novoNome } : g)));
  };

  const updateGastoValor = (id, novoValor) => {
    setGastos(gastos.map((g) => (g.id === id ? { ...g, valor: novoValor === "" ? 0 : Number(novoValor) } : g)));
  };

  return (
    <div className="root">
      <style>{CSS}</style>

      {!ready ? (
        <div className="loading">Carregando…</div>
      ) : (
        <div className="shell">
          <header className="header">Controle de Gastos</header>

          <div className="card saldo-card">
            <div className="saldo-block">
              <div className="saldo-label">Total de saídas</div>
              <div className="saldo-value out">{money(totalGastos)}</div>
            </div>
            <div className="divider" />
            <div className="saldo-block">
              <div className="saldo-label">Saldo (entrada − saídas)</div>
              <div className={`saldo-value ${saldo < 0 ? "out" : "in"}`}>{money(saldo)}</div>
            </div>
          </div>

          <div className="card">
            <label className="label">Valor que eu tenho (entrada manual)</label>
            <div className="input-wrap">
              <span className="prefix">R$</span>
              <input
                className="input-plain"
                type="number"
                inputMode="decimal"
                placeholder="0,00"
                value={entrada}
                onChange={(e) => setEntrada(e.target.value)}
              />
            </div>
          </div>

          <div className="card">
            <label className="label">Adicionar gasto</label>
            <div className="add-row">
              <input
                className="input-field grow"
                type="text"
                placeholder="Nome do gasto"
                value={nome}
                onChange={(e) => setNome(e.target.value)}
                onKeyDown={(e) => e.key === "Enter" && addGasto()}
              />
              <input
                className="input-field amount-w"
                type="number"
                inputMode="decimal"
                placeholder="Valor"
                value={valor}
                onChange={(e) => setValor(e.target.value)}
                onKeyDown={(e) => e.key === "Enter" && addGasto()}
              />
              <button className="add-btn" onClick={addGasto} disabled={!canAdd} aria-label="Adicionar">
                <Plus size={18} />
              </button>
            </div>
          </div>

          <div className="list-title">
            Gastos {gastos.length > 0 ? `(${gastos.length})` : ""}
          </div>

          {gastos.length === 0 ? (
            <div className="empty">Nenhum gasto lançado ainda.</div>
          ) : (
            <div className="list">
              {gastos.map((g) => (
                <div className="row" key={g.id}>
                  <input
                    className="row-nome-input"
                    type="text"
                    value={g.nome}
                    onChange={(e) => updateGastoNome(g.id, e.target.value)}
                  />
                  <div className="row-valor-wrap">
                    <span className="row-valor-prefix">R$</span>
                    <input
                      className="row-valor-input"
                      type="number"
                      inputMode="decimal"
                      value={g.valor}
                      onChange={(e) => updateGastoValor(g.id, e.target.value)}
                    />
                  </div>
                  <button className="del-btn" onClick={() => removeGasto(g.id)} aria-label="Remover">
                    <Trash2 size={15} />
                  </button>
                </div>
              ))}
            </div>
          )}
        </div>
      )}
    </div>
  );
}

const CSS = `
@import url('https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,600&family=IBM+Plex+Mono:wght@400;500&family=Inter:wght@400;500;600&display=swap');

* { box-sizing: border-box; }

.root { background: #0F1B2D; min-height: 100vh; font-family: 'Inter', sans-serif; color: #E8E4DA; }
.loading { display:flex; align-items:center; justify-content:center; min-height:100vh; color:#8B96A8; font-size:13px; }

.shell { max-width: 460px; margin: 0 auto; padding: 22px 18px 40px; display: flex; flex-direction: column; gap: 14px; }

.header { font-family: 'Fraunces', serif; font-size: 20px; margin-bottom: 4px; }

.card {
  background: #16243A;
  border: 1px solid #2A3B54;
  border-radius: 12px;
  padding: 16px;
}

.saldo-card { display: flex; align-items: center; }
.saldo-block { flex: 1; }
.divider { width: 1px; align-self: stretch; background: #2A3B54; margin: 0 14px; }
.saldo-label { font-size: 10.5px; text-transform: uppercase; letter-spacing: 0.8px; color: #8B96A8; margin-bottom: 6px; }
.saldo-value { font-family: 'IBM Plex Mono', monospace; font-size: 19px; font-weight: 500; }
.saldo-value.out { color: #C1584A; }
.saldo-value.in { color: #C9A667; }

.label { font-size: 11px; text-transform: uppercase; letter-spacing: 0.7px; color: #8B96A8; display:block; margin-bottom: 8px; }

.input-wrap { display: flex; align-items: center; gap: 8px; background: #1E2E47; border: 1px solid #2A3B54; border-radius: 9px; padding: 10px 13px; }
.prefix { color: #8B96A8; font-family: 'IBM Plex Mono', monospace; font-size: 14px; }
.input-plain { flex: 1; background: transparent; border: none; outline: none; color: #E8E4DA; font-family: 'IBM Plex Mono', monospace; font-size: 17px; }

.add-row { display: flex; gap: 8px; }
.input-field {
  background: #1E2E47; border: 1px solid #2A3B54; border-radius: 9px;
  padding: 10px 12px; color: #E8E4DA; font-size: 14px; font-family: 'Inter', sans-serif; outline: none;
}
.input-field:focus { outline: 2px solid #C9A667; outline-offset: 1px; }
.grow { flex: 1; min-width: 0; }
.amount-w { width: 100px; }
.add-btn {
  background: #C9A667; border: none; border-radius: 9px; width: 42px; flex-shrink: 0;
  color: #0F1B2D; display: flex; align-items: center; justify-content: center; cursor: pointer;
}
.add-btn:disabled { opacity: 0.35; cursor: not-allowed; }

.list-title { font-size: 11px; text-transform: uppercase; letter-spacing: 0.8px; color: #8B96A8; margin-top: 6px; }
.empty { color: #8B96A8; font-size: 13px; padding: 6px 2px; }

.list { display: flex; flex-direction: column; gap: 6px; }
.row {
  display: flex; align-items: center; gap: 10px;
  background: #16243A; border: 1px solid #2A3B54; border-radius: 9px; padding: 11px 13px;
}
.row-nome-input {
  flex: 1; min-width: 0; font-size: 13.5px; font-family: 'Inter', sans-serif; color: #E8E4DA;
  background: transparent; border: none; outline: none; padding: 4px 2px; border-radius: 5px;
}
.row-nome-input:focus { background: #1E2E47; }
.row-valor-wrap { display: flex; align-items: center; gap: 3px; flex-shrink: 0; }
.row-valor-prefix { font-family: 'IBM Plex Mono', monospace; font-size: 12px; color: #C1584A; opacity: 0.8; }
.row-valor-input {
  width: 74px; text-align: right; font-family: 'IBM Plex Mono', monospace; font-size: 13px; color: #C1584A;
  background: transparent; border: none; outline: none; padding: 4px 2px; border-radius: 5px;
}
.row-valor-input:focus { background: #1E2E47; }
.row-valor-input::-webkit-inner-spin-button, .row-valor-input::-webkit-outer-spin-button { -webkit-appearance: none; margin: 0; }
.del-btn { background: transparent; border: none; color: #8B96A8; cursor: pointer; padding: 3px; flex-shrink: 0; }
.del-btn:hover { color: #C1584A; }
`;
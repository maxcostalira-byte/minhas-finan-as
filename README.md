[app (2).py](https://github.com/user-attachments/files/24381943/app.2.py)
import streamlit as st
import pandas as pd
import plotly.express as px
import os
from datetime import date

# Configuração da página
st.set_page_config(page_title="Controle Financeiro Profissional", layout="wide", page_icon="💰")

# Caminhos dos arquivos de dados
DATA_FILE = "dados_financeiros.csv"
BUDGET_FILE = "metas_orcamento.csv"

# -----------------------------
# Categorias padrão
# -----------------------------
CATEGORIAS = {
    "Entrada": [
        "Salário / Pró-labore", "Vendas / Serviços", "Comissões", 
        "Rendimentos / Investimentos", "Aluguéis", "Reembolsos", "Outros ganhos"
    ],
    "Saída": [
        "Aluguel / Moradia", "Energia elétrica", "Água", "Internet / Telefonia", 
        "Condomínio", "Escola / Educação", "Plano de saúde", "Seguros", 
        "Assinaturas / Softwares", "Alimentação", "Transporte", "Combustível", 
        "Lazer", "Vestuário", "Compras diversas", "Saúde / Farmácia", 
        "Manutenção", "Impostos", "Cartão de crédito", "Empréstimos / Financiamentos", 
        "Parcelamentos", "Juros / Taxas bancárias"
    ]
}

FORMAS_PAGAMENTO = ["Pix", "Crédito", "Débito", "Dinheiro", "Transferência", "Boleto"]

# -----------------------------
# Funções de Dados
# -----------------------------
def carregar_dados():
    if os.path.exists(DATA_FILE):
        try:
            df = pd.read_csv(DATA_FILE)
            df["Data"] = pd.to_datetime(df["Data"])
            return df
        except:
            pass
    return pd.DataFrame(columns=["Data", "Tipo", "Categoria", "Descrição", "Valor", "Forma", "Observações"])

def carregar_metas():
    if os.path.exists(BUDGET_FILE):
        try:
            return pd.read_csv(BUDGET_FILE)
        except:
            pass
    return pd.DataFrame(columns=["Categoria", "Meta"])

def salvar_dados(df, file):
    df.to_csv(file, index=False)

# Inicialização do estado
if "dados" not in st.session_state:
    st.session_state.dados = carregar_dados()
if "metas" not in st.session_state:
    st.session_state.metas = carregar_metas()

# -----------------------------
# Sidebar: Adicionar Lançamento (Sem st.form para permitir atualização dinâmica)
# -----------------------------
st.sidebar.header("➕ Novo Lançamento")

data_lanc = st.sidebar.date_input("Data", value=date.today())
tipo = st.sidebar.selectbox("Tipo de Movimentação", ["Entrada", "Saída"])

# Agora a categoria atualiza IMEDIATAMENTE quando o tipo muda
categorias_opcoes = CATEGORIAS[tipo]
categoria = st.sidebar.selectbox("Categoria", categorias_opcoes)

descricao = st.sidebar.text_input("Descrição Detalhada")
valor = st.sidebar.number_input("Valor (R$)", min_value=0.0, step=10.0, format="%.2f")
forma = st.sidebar.selectbox("Forma de Pagamento", FORMAS_PAGAMENTO)
obs = st.sidebar.text_area("Observações (opcional)", height=70)

if st.sidebar.button("Registrar Movimentação", type="primary"):
    if not descricao.strip():
        st.sidebar.error("Por favor, preencha a descrição.")
    elif valor <= 0:
        st.sidebar.error("O valor deve ser maior que zero.")
    else:
        novo_lancamento = pd.DataFrame([{
            "Data": pd.to_datetime(data_lanc), "Tipo": tipo, "Categoria": categoria,
            "Descrição": descricao.strip(), "Valor": float(valor), "Forma": forma, "Observações": obs.strip()
        }])
        st.session_state.dados = pd.concat([st.session_state.dados, novo_lancamento], ignore_index=True)
        salvar_dados(st.session_state.dados, DATA_FILE)
        st.sidebar.success("Lançamento registrado!")
        st.rerun()

# -----------------------------
# Interface Principal com Abas
# -----------------------------
st.title("📊 Gestão Financeira Inteligente")

tab_lancamentos, tab_resumo, tab_orcamento, tab_graficos = st.tabs([
    "📝 LANÇAMENTOS", "📅 RESUMO MENSAL", "🎯 ORÇAMENTO", "📈 GRÁFICOS"
])

df = st.session_state.dados.copy()
if not df.empty:
    df["Data"] = pd.to_datetime(df["Data"])
    df["Mês/Ano"] = df["Data"].dt.strftime('%m/%Y')

# --- ABA: LANÇAMENTOS ---
with tab_lancamentos:
    st.subheader("Histórico de Movimentações")
    if df.empty:
        st.info("Nenhum dado registrado ainda.")
    else:
        col_f1, col_f2, col_f3 = st.columns(3)
        with col_f1: filtro_tipo = st.multiselect("Tipo", ["Entrada", "Saída"], default=["Entrada", "Saída"])
        with col_f2: 
            lista_cats = sorted(list(set(df["Categoria"])))
            filtro_cat = st.multiselect("Categoria", lista_cats, default=lista_cats)
        with col_f3:
            data_min, data_max = df["Data"].min().date(), df["Data"].max().date()
            periodo = st.date_input("Período", value=(data_min, data_max))
        
        df_filtrado = df[(df["Tipo"].isin(filtro_tipo)) & (df["Categoria"].isin(filtro_cat))]
        if len(periodo) == 2:
            df_filtrado = df_filtrado[(df_filtrado["Data"].dt.date >= periodo[0]) & (df_filtrado["Data"].dt.date <= periodo[1])]
        
        st.dataframe(df_filtrado.sort_values("Data", ascending=False), use_container_width=True, hide_index=True,
                     column_config={"Valor": st.column_config.NumberColumn("Valor (R$)", format="R$ %.2f"),
                                    "Data": st.column_config.DateColumn("Data", format="DD/MM/YYYY")})
        
        if st.button("🗑️ Limpar Todos os Dados"):
            if st.checkbox("Confirmar exclusão permanente?"):
                st.session_state.dados = pd.DataFrame(columns=["Data", "Tipo", "Categoria", "Descrição", "Valor", "Forma", "Observações"])
                salvar_dados(st.session_state.dados, DATA_FILE)
                st.rerun()

# --- ABA: RESUMO MENSAL ---
with tab_resumo:
    st.subheader("Relatório de Performance Mensal")
    if df.empty:
        st.warning("Adicione lançamentos para visualizar o resumo.")
    else:
        total_entradas = df[df["Tipo"] == "Entrada"]["Valor"].sum()
        total_saidas = df[df["Tipo"] == "Saída"]["Valor"].sum()
        saldo_geral = total_entradas - total_saidas
        
        m1, m2, m3 = st.columns(3)
        m1.metric("Total Entradas", f"R$ {total_entradas:,.2f}")
        m2.metric("Total Saídas", f"R$ {total_saidas:,.2f}", delta_color="inverse")
        m3.metric("Saldo Acumulado", f"R$ {saldo_geral:,.2f}")

        resumo_mensal = df.groupby(["Mês/Ano", "Tipo"])["Valor"].sum().unstack(fill_value=0).reset_index()
        for col in ["Entrada", "Saída"]:
            if col not in resumo_mensal.columns: resumo_mensal[col] = 0
        resumo_mensal["Saldo"] = resumo_mensal["Entrada"] - resumo_mensal["Saída"]
        st.dataframe(resumo_mensal, use_container_width=True, hide_index=True,
                     column_config={c: st.column_config.NumberColumn(format="R$ %.2f") for c in ["Entrada", "Saída", "Saldo"]})

# --- ABA: ORÇAMENTO ---
with tab_orcamento:
    st.subheader("🎯 Planejamento de Metas de Gastos")
    col_met1, col_met2 = st.columns([1, 2])
    with col_met1:
        st.write("**Definir Metas**")
        with st.form("form_metas"):
            cat_meta = st.selectbox("Categoria de Saída", CATEGORIAS["Saída"])
            valor_meta = st.number_input("Meta Mensal (R$)", min_value=0.0, step=50.0)
            if st.form_submit_button("Salvar Meta"):
                metas_atuais = st.session_state.metas
                if cat_meta in metas_atuais["Categoria"].values:
                    metas_atuais.loc[metas_atuais["Categoria"] == cat_meta, "Meta"] = valor_meta
                else:
                    metas_atuais = pd.concat([metas_atuais, pd.DataFrame([{"Categoria": cat_meta, "Meta": valor_meta}])], ignore_index=True)
                st.session_state.metas = metas_atuais
                salvar_dados(metas_atuais, BUDGET_FILE)
                st.success(f"Meta para {cat_meta} atualizada!")
                st.rerun()
    with col_met2:
        st.write("**Acompanhamento de Metas (Mês Atual)**")
        mes_atual = date.today().strftime('%m/%Y')
        gastos_mes = df[(df["Tipo"] == "Saída") & (df["Mês/Ano"] == mes_atual)].groupby("Categoria")["Valor"].sum().reset_index()
        if st.session_state.metas.empty:
            st.info("Defina metas para acompanhar seu orçamento.")
        else:
            comp_orcamento = pd.merge(st.session_state.metas, gastos_mes, on="Categoria", how="left").fillna(0)
            comp_orcamento.columns = ["Categoria", "Meta (R$)", "Gasto Real (R$)"]
            comp_orcamento["Diferença"] = comp_orcamento["Meta (R$)"] - comp_orcamento["Gasto Real (R$)"]
            comp_orcamento["Status"] = comp_orcamento["Diferença"].apply(lambda x: "✅ No Prazo" if x >= 0 else "🚨 Excedido")
            st.dataframe(comp_orcamento, use_container_width=True, hide_index=True,
                         column_config={"Meta (R$)": st.column_config.NumberColumn(format="R$ %.2f"),
                                        "Gasto Real (R$)": st.column_config.NumberColumn(format="R$ %.2f"),
                                        "Diferença": st.column_config.NumberColumn(format="R$ %.2f")})
            fig_orc = px.bar(comp_orcamento, x="Categoria", y=["Meta (R$)", "Gasto Real (R$)"], 
                             barmode="group", title="Meta vs Real por Categoria",
                             color_discrete_map={"Meta (R$)": "#bdc3c7", "Gasto Real (R$)": "#e74c3c"})
            st.plotly_chart(fig_orc, use_container_width=True)

# --- ABA: GRÁFICOS ---
with tab_graficos:
    st.subheader("Visualização de Dados")
    if df.empty:
        st.info("Gráficos serão exibidos aqui após o registro de dados.")
    else:
        col_g1, col_g2 = st.columns(2)
        with col_g1:
            fig_bar = px.bar(df.groupby(["Mês/Ano", "Tipo"])["Valor"].sum().reset_index(),
                             x="Mês/Ano", y="Valor", color="Tipo", barmode="group",
                             color_discrete_map={"Entrada": "#2ecc71", "Saída": "#e74c3c"})
            st.plotly_chart(fig_bar, use_container_width=True)
        with col_g2:
            df_saidas = df[df["Tipo"] == "Saída"]
            if not df_saidas.empty:
                fig_pie = px.pie(df_saidas, values="Valor", names="Categoria", hole=0.4)
                st.plotly_chart(fig_pie, use_container_width=True)
        df_evolucao = df.sort_values("Data")
        df_evolucao["Valor_Ajustado"] = df_evolucao.apply(lambda x: x["Valor"] if x["Tipo"] == "Entrada" else -x["Valor"], axis=1)
        df_evolucao["Saldo_Acumulado"] = df_evolucao["Valor_Ajustado"].cumsum()
        fig_line = px.line(df_evolucao, x="Data", y="Saldo_Acumulado", title="Evolução do Saldo")
        st.plotly_chart(fig_line, use_container_width=True)

st.sidebar.divider()
st.sidebar.caption("Controle Financeiro Inteligente.")

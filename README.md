# prescriptor
import streamlit as st
import pandas as pd
import re
import os

# =====================
# Load / Prepare Data
# =====================

INTERACTIONS_CSV = "data/drug_interactions.csv"
ALTS_CSV = "data/drug_alternatives.csv"

if not os.path.exists("data"):
    os.makedirs("data")

# Create sample datasets if missing
if not os.path.exists(INTERACTIONS_CSV):
    with open(INTERACTIONS_CSV, "w") as f:
        f.write("drug_a,drug_b,severity,notes\n"
                "aspirin,warfarin,high,Increased bleeding ri
                "ibuprofen,warfarin,moderate,Increased bleed
                "lisinopril,potassium_supplements,high,Hyper
                "metformin,contrast_media,high,Lactic acidos
                "amoxicillin,warfarin,low,Minor interaction\

if not os.path.exists(ALTS_CSV):
    with open(ALTS_CSV, "w") as f:
        f.write("drug,alternative,reason\n"
                "aspirin,acetaminophen,Lower bleeding risk\n
                "ibuprofen,naproxen,Longer duration\n"
                "lisinopril,amlodipine,Different mechanism (
                "warfarin,apixaban,Direct oral anticoagulant

interactions_df = pd.read_csv(INTERACTIONS_CSV)
alts_df = pd.read_csv(ALTS_CSV)


# =====================
# Helper Functions
# =====================

def simple_extract_drugs(text: str):
    """Extracts drugs and dosages from free text using regex
    rx = re.findall(r"([A-Za-z0-9_\-]+)\s*([0-9]+\s*(?:mg|ml
    results = []
    for match in rx:
        name = match[0].lower()
        dosage = match[1] if match[1] else None
        results.append({'name': name, 'dosage': dosage})
    return results


def check_interactions(drugs):
    """Check for drug-drug interactions from CSV."""
    findings = []
    for i in range(len(drugs)):
        for j in range(i + 1, len(drugs)):
            a, b = drugs[i], drugs[j]
            row = interactions_df[((interactions_df['drug_a'
                                  ((interactions_df['drug_a'
            if not row.empty:
                for _, r in row.iterrows():
                    findings.append(dict(r))
    return findings


def suggest_alternatives(drugs):
    """Suggest safer alternatives from CSV."""
    suggestions = []
    for n in drugs:
        row = alts_df[alts_df['drug'] == n]
        if not row.empty:
            for _, r in row.iterrows():
                suggestions.append(dict(r))
    return suggestions


def dosage_recommendation(drugs, age):
    """Rule-based dosage recommendations (demo)."""
    res = []
    for name in drugs:
        if 'aspirin' in name:
            rec = "Not recommended for children (Reye's risk
        elif 'paracetamol' in name or 'acetaminophen' in nam
            rec = "Child: 10–15 mg/kg q4–6h, max 75 mg/kg/da
        elif 'ibuprofen' in name:
            rec = "Child: 5–10 mg/kg q6–8h, max 40 mg/kg/day
        else:
            rec = "No rule-based dosage available in demo."
        res.append({'drug': name, 'recommendation': rec})
    return res


# =====================
# Streamlit UI
# =====================

st.set_page_config(page_title="Drug Interaction & Dosage Ass

st.title("💊 AI Medical Prescriptor (Demo Prototype)")

mode = st.radio('Choose input method:', ['Free text', 'Manua

if mode == 'Free text':
    text = st.text_area('Paste prescription / clinical note 
    if st.button('Extract drugs'):
        extracted = simple_extract_drugs(text)
        st.session_state['last_extracted'] = extracted
        st.subheader('Extracted Drugs')
        if extracted:
            for d in extracted:
                st.write(f"- {d['name']} {d['dosage'] or ''}
        else:
            st.info("No drugs detected. Try entering drug na

else:
    manual = st.text_area('Enter one drug per line (e.g., "a
    if st.button('Use manual list'):
        items = []
        for line in manual.splitlines():
            if not line.strip():
                continue
            parts = [p.strip() for p in line.split(',')]
            items.append({'name': parts[0].lower(), 'dosage'
        st.session_state['last_extracted'] = items
        st.success('Saved manual drug list')

age = st.number_input('Enter patient age (years):', min_valu

if st.button('Analyze prescription'):
    drugs = [d['name'] for d in st.session_state.get('last_e
    if not drugs:
        st.warning('⚠️ No drugs to analyze.')
    else:
        interactions = check_interactions(drugs)
        dosages = dosage_recommendation(drugs, int(age))
        alts = suggest_alternatives(drugs)

        # Interactions
        st.subheader('🚨 Drug Interactions')
        if interactions:
            for it in interactions:
                st.error(f"{it['drug_a']} + {it['drug_b']}: 
        else:
            st.success('✅ No known interactions (from demo 

        # Dosages
        st.subheader('💉 Dosage Recommendations')
        for d in dosages:
            st.write(f"- **{d['drug']}**: {d['recommendation

        # Alternatives
        st.subheader('🔄 Alternative Suggestions')
        if alts:
            for a in alts:
                st.write(f"- {a['drug']} → {a['alternative']
        else:
            st.info('No alternatives found in demo DB.')

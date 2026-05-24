# Findings Summary

A two-page version of the report. For the full 25-page writeup with citations, see [`report/final_report.pdf`](../report/final_report.pdf).

---

## Sample

After filtering to Southern Maine, **n = 1,702 respondents** across Cumberland County (65%), York County (30%), and both (4%). About 30% are Below the ALICE threshold (struggling to afford basic local costs despite earning above the federal poverty line). 19% of respondents who answered the question say they want to get involved.

---

## Six headline findings

**1. Housing cost is a cross-class crisis.** 82% of respondents cite housing as a biggest challenge. Below-ALICE and Above-ALICE respondents cite it at essentially the same rate (68.8% vs 65.2%). Housing stress is the ambient condition of community life, not a poverty marker. Programmes framed exclusively as ALICE intervention will structurally exclude most affected residents.

**2. Below and Above ALICE households face structurally different problems.** Below-ALICE respondents cite survival items more (food +10pp, transportation +14pp, basics +9pp). Above-ALICE respondents cite infrastructure-access items more (child care +12pp, elder care +8pp, healthcare cost +10pp). Three methodologically independent analyses — chi-square, logistic regression, and decision tree — identify the same diagnostic challenge set.

**3. The benefit cliff is visible in the data.** Child care and elder care are cited more by Above-ALICE respondents — not because Below-ALICE households don't need those services, but because they're often eligible for subsidies (Head Start, Medicaid elder care) that Above-ALICE households are priced out of while still being unable to afford market-rate care.

**4. Challenge profiles can identify ALICE-status signal but cannot deploy as a classifier.** The class-balanced logistic regression achieves F1 = 0.466 against a 30/70 class split. That's well above chance but well below deployment quality. The result is substantively informative: challenges capture *lived experience*, but income classification needs structural variables (savings, debt, household composition) absent from the instrument.

**5. The community that most often feels unheard is a place, not a demographic.** Free-text responses to "which community feels unheard?" were processed through a spaCy NER + zero-shot classification pipeline. After classification, the single largest unheard-community label is **town of residence** (40.5% of identified communities), outpacing BIPOC (13.6%), women (9.1%), and older adults (8.3%) combined. Hyperlocal civic engagement may matter as much as demographic outreach.

**6. Feeling unheard does not predict engagement willingness.** About 37% of respondents who answered report feeling unheard, but only 19% say they want to get involved. Chi-square tests show that engagement willingness is significantly tied to race/ethnicity (Cramér's V = 0.224) and ALICE status (V = 0.172), but not to feeling heard (p > 0.05). Below-ALICE willing respondents prefer to contribute time, voice, and lived experience (0% donate); Above-ALICE willing respondents are willing to contribute financially in addition.

---

## What to take away

Southern Maine has a **layered hardship landscape**: a universal top layer of housing and mental health stress that cuts across income, a survival layer specific to Below-ALICE households, and an infrastructure-access layer specific to Above-ALICE households caught in the benefit cliff. Underneath the material story is a civic story patterned by economic status but not explained by income alone — geography drives perceived exclusion as powerfully as demographics, and willingness to act is shaped more by racial and cultural identity than by income.

For UWSM, this implies a portfolio rather than a single programme: treat housing as the universal crisis it is, bundle survival services for Below-ALICE households, build bridge programmes for Above-ALICE households caught in the benefit cliff, invest in hyperlocal civic infrastructure at the town level, and design engagement strategies that are simultaneously income-sensitive and culturally specific.

---

## What the data can't tell us

These findings should be read with three caveats. **Demographic missingness is structural** (58% of respondents took an abbreviated instrument that didn't ask race, age, or feel-heard), so subgroup analyses are restricted to the Full and ALICE Voices instruments. **The BIPOC subsample is n = 30**, too small for statistically reliable race-based inference. **All findings are associational** — no result supports a causal claim.

The full discussion of limitations and recommendations for the next round of data collection is in Section 6 of the report.

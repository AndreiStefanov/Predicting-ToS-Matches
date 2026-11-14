# Predicting Trials of Osiris Match Outcomes


# Project Overview

This repository contains the Phase 1 specification, data warehousing schema, and methodological framework for predicting match outcomes in Destiny 2's competitive Trials of Osiris (3v3) game mode.

Standard match prediction models often rely entirely on cumulative ELO or lifetime win-rate ratings. This project explores a more granular approach: predicting match outcomes using team class composition, weapon archetypes, map context, and team synergy, while isolating raw player skill into a controlled ablation study.

---

# Key Challenges Addressed

1. Avoiding ELO / Win-Rate Padding
   * Problem: Traditional machine learning approaches for Destiny 2 rely heavily on historical win-rate slices (last 10 matches, last 30 matches, standard deviations), which essentially re-invent an ELO rating system.
   * Solution: This project pivots to tactical composition parameters.

2. The Target Leakage Problem
   * Problem: Utilizing post-match weapon kills directly from the Post-Game Carnage Report (PGCR) introduces target leakage (winning teams record more kills and weapons).
   * Solution: Our schema addresses this by mapping tactical archetypes and expected loadout distribution.

3. Lobby Filtering (Data Hygiene)
   * Problem: Raw scrapers capture extreme blowout matches (e.g., a top 0.1% 3-stack vs. solo-queue novices) where raw mechanical skill entirely dominates the outcome.
   * Solution: We implement a strict skill-gap filter to focus training data on closely contested lobbies where loadout and composition actually matter.

---

# Feature Schema (15 Distinct Features)

Each row in the dataset represents a single unique 3v3 Trials of Osiris match. All input features (except the environment) are calculated as differences between Team A and Team B.

1. map_hash (Environment, Categorical) — The specific PvP map played (`activityDetails.referenceId`)
2. delta_warlocks (Class Comp, Integer) — Team A Warlock count minus Team B Warlock count
3. delta_hunters (Class Comp, Integer) — Team A Hunter count minus Team B Hunter count
4. delta_titans (Class Comp, Integer) — Team A Titan count minus Team B Titan count
5. delta_class_diversity (Class Comp, Integer) — Unique class count difference (tests stacking vs. balance)
6. delta_hand_cannons (Loadout - Primary, Integer) — Difference in Hand Cannon users (`itemSubType == 6`)
7. delta_pulse_scouts (Loadout - Primary, Integer) — Difference in long-range primary users (Pulse & Scout Rifles)
8. delta_auto_smg_sidearm (Loadout - Primary, Integer) — Difference in close-range automatic primary users
9. delta_snipers (Loadout - Special, Integer) — Difference in Sniper Rifle users (`itemSubType == 12`)
10. delta_shotguns_fusions (Loadout - Special, Integer) — Difference in close-quarters special weapon users
11. delta_exotic_weapons (Loadout - Tier, Integer) — Difference in Exotic-tier weapons deployed (`tierType == 6`)
12. delta_premade_clan_pairs (Team Synergy, Integer) — Difference in shared clan tags (indicates pre-made fireteams)
13. delta_ability_reliance (Playstyle, Float) — Difference in team ratio of ability kills to weapon kills
14. delta_precision_ratio (Playstyle, Float) — Difference in ratio of headshot kills to total kills
15. delta_recent_fireteam_matches (Team Synergy, Integer) — Frequency these 3 teammates have queued together prior

# Target Label & Control Variable

* target_A_wins (Target Label $Y$, Binary) — Binary target: `1` if Team A won, `0` if Team B won
* delta_team_kd (Auxiliary Control Variable, Float) — Difference in rolling team average K/D ratio, used exclusively for Model B control

---

# Data Warehousing & Collection Pipeline

* Source Data: Bungie API (`GetPostGameCarnageReport`) cross-referenced with the local SQLite Bungie Manifest (`DestinyInventoryItemDefinition` and `DestinyActivityDefinition`).
* Target Volume: Minimum 1,000+ valid, non-truncated 3v3 Trials matches.
* Hygiene Rule: Matches exhibiting extreme historical skill discrepancies (absolute K/D delta exceeding safety thresholds) are filtered out to maintain predictive variance in loadout analysis.

---

# Machine Learning & Experiment Design (Ablation Study)

To scientifically evaluate whether loadout and composition dictate match outcomes independently of player skill, we benchmark two models:

* Model A (Pure Composition & Loadout): Uses only the 15 class, weapon, map, and synergy features. Excludes any skill metrics entirely.
* Model B (Composition + Skill Control): Uses the exact same 15 features plus `delta_team_kd`.

# Selected Algorithms for Benchmarking

* Random Forest Classifier: Combines multiple decision trees and handles non-linear feature interactions robustly (e.g., map-to-weapon synergies).
* XGBoost Classifier: Sequential gradient boosting framework to test whether error-correcting boosting improves predictive performance over Random Forest.
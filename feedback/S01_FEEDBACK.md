# Rétroaction automatisée -- S01 (Diagnostic fondamental -- NexaMart kickoff)

_Générée le 2026-05-15T12:40:20+00:00 -- Run `20260515T122624Z-00a5a04f`_

Ce document est produit par un pipeline reproductible (vérification SQL déterministe + analyse LLM du brief et de la déclaration IA). Une revue humaine précède toujours sa publication. **À ce stade expérimental, aucune note ni étiquette de niveau n'est diffusée : l'objectif est purement formatif.**

> ⚠️ **Avertissement instructeur (à retirer avant publication) :** cette analyse a été générée avec `--skip-pull`. Le contenu correspond au commit local et **n'est peut-être pas la dernière version poussée par l'étudiant·e**.

---

## 1. Vérification automatique de la requête SQL

La requête extraite de votre brief n'a pas pu être validée automatiquement. Quelques pistes constructives ci-dessous pour vous aider à la rendre exécutable et alignee avec la question posée.

_Observation technique : erreur d'exécution SQL: Conversion Error: invalid date field format: "[REDACTED-PHONE]", expected format is (YYYY-MM-DD)_

<details><summary>Requête analysée — cliquez pour déplier</summary>

```sql
WITH sales_by_half AS (
    SELECT
        p.category,
        st.region,
        st.province,
        CASE
            WHEN s.order_date < DATE '[REDACTED-PHONE]' THEN 'S1'
            ELSE 'S2'
        END AS semestre,
        SUM(s.line_total) AS revenue,
        SUM(s.quantity) AS units,
        AVG(s.discount_pct) AS avg_discount
    FROM raw_fact_sales s
    JOIN raw_dim_product p
        ON s.product_id = p.product_id
    JOIN raw_dim_store st
        ON s.store_id = st.store_id
    WHERE s.order_date BETWEEN DATE '[REDACTED-PHONE]' AND DATE '[REDACTED-PHONE]'
    GROUP BY
        p.category,
        st.region,
        st.province,
        semestre
),
declining_categories AS (
    SELECT
        category,
        region,
        province,
        SUM(CASE WHEN semestre = 'S1' THEN revenue ELSE 0 END) AS revenue_s1,
        SUM(CASE WHEN semestre = 'S2' THEN revenue ELSE 0 END) AS revenue_s2,
        SUM(CASE WHEN semestre = 'S1' THEN units ELSE 0 END) AS units_s1,
        SUM(CASE WHEN semestre = 'S2' THEN units ELSE 0 END) AS units_s2,
        AVG(CASE WHEN semestre = 'S1' THEN avg_discount END) AS discount_s1,
        AVG(CASE WHEN semestre = 'S2' THEN avg_discount END) AS discount_s2
    FROM sales_by_half
    GROUP BY
        category,
        region,
        province
    HAVING
        SUM(CASE WHEN semestre = 'S2' THEN revenue ELSE 0 END)
        <
        SUM(CASE WHEN semestre = 'S1' THEN revenue ELSE 0 END)
),
returns_s2 AS (
    SELECT
        p.category,
        st.region,
        st.province,
        SUM(r.refund_amount) AS refund_amount_s2,
        SUM(r.return_quantity) AS return_units_s2,
        COUNT(*) AS return_lines_s2,
        MODE(r.return_reason) AS main_return_reason
    FROM raw_fact_returns r
    JOIN raw_dim_product p
        ON r.product_id = p.product_id
    JOIN raw_dim_store st
        ON r.store_id = st.store_id
    WHERE r.return_date BETWEEN DATE '[REDACTED-PHONE]' AND DATE '[REDACTED-PHONE]'
    GROUP BY
        p.category,
        st.region,
        st.province
),
inventory_s2 AS (
    SELECT
        p.category,
        st.region,
        st.province,
        AVG(
            CASE
                WHEN i.quantity_on_hand <= i.reorder_point THEN 1.0
                ELSE 0.0
            END
        ) AS low_stock_rate_s2
    FROM raw_fact_inventory_snapshot i
    JOIN raw_dim_product p
        ON i.product_id = p.product_id
    JOIN raw_dim_store st
        ON i.store_id = st.store_id
    WHERE i.snapshot_date BETWEEN DATE '[REDACTED-PHONE]' AND DATE '[REDACTED-PHONE]'
    GROUP BY
        p.category,
        st.region,
        st.province
)
SELECT
    d.category,
    d.region,
    d.province,

    ROUND(d.revenue_s1, 2) AS revenue_s1,
    ROUND(d.revenue_s2, 2) AS revenue_s2,
    ROUND(d.revenue_s2 - d.revenue_s1, 2) AS revenue_change,
    ROUND(100 * (d.revenue_s2 - d.revenue_s1) / NULLIF(d.revenue_s1, 0), 1) AS pct_change,

    d.units_s1,
    d.units_s2,
    d.units_s2 - d.units_s1 AS unit_change,

    ROUND(100 * COALESCE(r.refund_amount_s2, 0) / NULLIF(d.revenue_s2, 0), 1) AS refund_rate_pct,
    COALESCE(r.main_return_reason, '-') AS main_return_reason,

    ROUND(100 * COALESCE(i.low_stock_rate_s2, 0), 1) AS low_stock_pct,

    ROUND(d.discount_s1 * 100, 1) AS discount_s1_pct,
    ROUND(d.discount_s2 * 100, 1) AS discount_s2_pct

FROM declining_categories d
LEFT JOIN returns_s2 r
    ON d.category = r.category
   AND d.region = r.region
   AND d.province = r.province
LEFT JOIN inventory_s2 i
    ON d.category = i.category
   AND d.region = i.region
   AND d.province = i.province
ORDER BY
    revenue_change ASC;
```

</details>


**Pistes :**
> Requête extraite par LLM (aucun bloc fencé détecté). Encadrez votre requête finale par ```sql ... ``` pour éliminer toute ambiguïté.
> Tables référencées dans votre requête mais absentes de la base : `declining_categories`, `inventory_s2`, `returns_s2`, `sales_by_half`.
> Tables disponibles dans `db/nexamart.duckdb` : `raw_bridge_campaign_allocation`, `raw_bridge_customer_segment`, `raw_customer_changes`, `raw_customer_profile_bands`, `raw_customer_scd3_history`, `raw_dim_channel`, `raw_dim_customer`, `raw_dim_date`, `raw_dim_geography`, `raw_dim_product`, `raw_dim_segment_outrigger`, `raw_dim_store`, `raw_fact_budget`, `raw_fact_daily_inventory`, `raw_fact_inventory_snapshot`, `raw_fact_order_pipeline`, `raw_fact_orders_transaction`, `raw_fact_promo_exposure`, `raw_fact_returns`, `raw_fact_sales`.

## 2. Rétroaction pédagogique sur le brief

> Bon diagnostic opérationnel : le brief identifie catégories et régions en déclin, fournit SQL de validation et propose pistes d'investigation pertinentes. Pour monter au niveau 'excellent', formalisez le modèle (SCD, patterns Kimball), publiez un trace git reproductible et synthétisez une recommandation board claire et actionnable.

### Observations par dimension

**Model quality**
- Observation : L'étudiant précise le grain (« chaque ligne représente une vente »), propose fact_sales et les dimensions produit/magasin/client/canal/date et explique l'usage des mesures quantité, rabais et montant total.
- Piste d'amélioration : Formaliser le pattern SCD attendu (ex. SCD Type 2 sur dim_product) et expliciter les choix de bridge/junk si pertinents (ex. customer segment bridge) pour corriger les risques historiques mentionnés.

**Validation quality**
- Observation : Le brief contient des requêtes SQL de validation, une réconciliation des totaux (2 577 ventes, 543 904,80 $) et des contrôles sur retours et inventaire.
- Piste d'amélioration : Ajouter des checks explicites pour les cas limites (NULLs, divisions par zéro via NULLIF, vérification d'unicité du grain) et fournir les résultats d'exécution systématiques (outputs de make check).

**Executive justification**
- Observation : La section ‘Réponse exécutive’ identifie catégories et régions prioritaires (ex. Home & Garden Alberta, Pet Supplies Québec) et recommande d'investiguer retours, ruptures et écarts budget/ventes.
- Piste d'amélioration : Rendre la recommandation au CEO plus décisionnelle et synthétique (p.ex. trois actions concrètes et prioritaires avec impact attendu et ETA), en réduisant le jargon technique.

**Process trace**
- Observation : L'étudiant décrit des actions locales (installation Python, génération de la DB, logs d'activité) et mentionne l'usage de ChatGPT mais n'inclut pas d'historique git avec commits détaillés.
- Piste d'amélioration : Publier un historique git clair (≥3 commits incrémentaux avec messages) et une note IA formelle précisant quel prompt/outil a produit quelles parties, plus la validation humaine.

**Reproducibility**
- Observation : L'auteur décrit les commandes utilisées (gen_all.py, run_pipeline) et affirme avoir créé db/nexamart.duckdb mais signale des étapes manuelles et des chemins locaux.
- Piste d'amélioration : Fournir un README minimal reproductible (clone → commandes exactes make generate/load/check) sans chemins codés en dur et avec un script d'initialisation automatisé.

## 3. Déclaration d'utilisation de l'IA

> La déclaration est détaillée et documente les interactions, les étapes et les validations humaines. Toutefois l'information sur l'outil reste incomplète (nom donné sans version/modèle précis), ce qui rend la déclaration partiellement générique.

**Sujets bien couverts dans votre déclaration :**

- outils utilisés (nom + version/modèle)
- à quelle étape l'IA a été utilisée
- comment la sortie a été validée par l'humain
- limites ou erreurs observées

## 4. Pistes d'action pour la prochaine itération

- Reprendre la requête de la section « Preuve » pour qu'elle s'exécute sur `db/nexamart.duckdb` et qu'elle produise la forme attendue (voir pistes en section 1).

---

## 5. Traçabilité

- **Run ID :** `20260515T122624Z-00a5a04f`
- **Devoir :** `S01`
- **Étudiant·e :** `jmpierre23`
- **Commit analysé :** `8e53c74`
- **Audit (côté instructeur) :** `tools/instructor/feedback_pipeline/audit/20260515T122624Z-00a5a04f/jmpierre23/`
- **Prompts (SHA-256) :**
  - `sql_extractor_system` : `90ee9e277de7a27f...`
  - `rubric_grader_system` : `505f32d1d8319d66...`
  - `ai_usage_grader_system` : `81cb7fdf89bda55a...`
- **Fournisseur (rubrique) :** `openai`
- **Fournisseur (IA-usage) :** `openai` (gpt-5-mini-2025-08-07)

_Ce feedback a été produit par un pipeline automatisé et **revu par l'équipe pédagogique avant publication**. Aucun chiffre ni étiquette de niveau n'est diffusé à ce stade expérimental : l'objectif est uniquement formatif. Ouvrez une issue dans ce dépôt pour toute question._

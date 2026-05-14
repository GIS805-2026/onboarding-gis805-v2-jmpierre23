# Rétroaction automatisée -- S01 (Diagnostic fondamental -- NexaMart kickoff)

_Générée le 2026-05-14T22:27:03+00:00 -- Run `20260514T221333Z-7d34bf6a`_

Ce document est produit par un pipeline reproductible (vérification SQL déterministe + analyse LLM du brief et de la déclaration IA). Une revue humaine précède toujours sa publication. **À ce stade expérimental, aucune note ni étiquette de niveau n'est diffusée : l'objectif est purement formatif.**

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
> Votre `db/nexamart.duckdb` est absente ou vide ; la requête a été exécutée contre une **base de référence cohorte** (seed instructeur). Les chiffres retournés ne correspondent donc pas à vos propres données : reconstruisez votre base avec `python src/run_pipeline.py` (ou `.\run.ps1 load`) pour valider vos calculs sur votre seed personnel.
> Tables référencées dans votre requête mais absentes de la base : `declining_categories`, `inventory_s2`, `returns_s2`, `sales_by_half`.
> Tables disponibles dans `db/nexamart.duckdb` : `raw_bridge_campaign_allocation`, `raw_bridge_customer_segment`, `raw_customer_changes`, `raw_customer_profile_bands`, `raw_customer_scd3_history`, `raw_dim_channel`, `raw_dim_customer`, `raw_dim_date`, `raw_dim_geography`, `raw_dim_product`, `raw_dim_segment_outrigger`, `raw_dim_store`, `raw_fact_budget`, `raw_fact_daily_inventory`, `raw_fact_inventory_snapshot`, `raw_fact_order_pipeline`, `raw_fact_orders_transaction`, `raw_fact_promo_exposure`, `raw_fact_returns`, `raw_fact_sales`.

## 2. Rétroaction pédagogique sur le brief

> Bon diagnostic opérationnel : l'étudiant a identifié les catégories et régions prioritaires et a réalisé de nombreuses vérifications sur les tables brutes. Il reste à formaliser le modèle (SCD, grain, DDL), à publier des checks reproductibles et à condenser le brief en une décision claire pour le CEO.

### Observations par dimension

**Model quality**
- Observation : On va créer une table principale, fact_sales, où chaque ligne représente une vente.
- Piste d'amélioration : Préciser le grain formel (clé composite), documenter explicitement le choix SCD (type) pour chaque dimension et noter les attributs non-additifs (ex. unit_price) avec exemples de DDL.

**Validation quality**
- Observation : J’ai commencé par vérifier que la base contenait bien toutes les tables nécessaires. La base contient 24 tables et aucune table principale n’est vide.
- Piste d'amélioration : Fournir les sorties exactes des contrôles (make check) : requêtes exécutées, résultats numériques et traitement des cas limites (NULLs, divisions par zéro, vérification du grain).

**Executive justification**
- Observation : Les priorités immédiates devraient être Home & Garden en Alberta, Pet Supplies au Québec, et plusieurs catégories en BC, Estrie et Outaouais.
- Piste d'amélioration : Rédiger une version condensée (150–300 mots) avec une recommandation décisionnelle claire (ex. actions prioritaires, KPI à surveiller, impact attendu) et chiffrée si possible.

**Process trace**
- Observation : La base DuckDB existe maintenant ici: db/nexamart.duckdb ... winget install Python.Python.3.12 ... j’ai fait: .\.venv\Scripts\python.exe -m pip install -r requirements.txt ...
- Piste d'amélioration : Ajouter un log git (≥3 commits) avec messages significatifs, et une note IA formelle (outil utilisé, prompt résumé, validation humaine) dans le decision log.

**Reproducibility**
- Observation : La base DuckDB existe maintenant ici: db/nexamart.duckdb et j’ai exécuté les commandes d’installation et de génération listées (winget, pip, gen_all.py, run_pipeline.py).
- Piste d'amélioration : Fournir un README reproductible : commandes exactes sans chemins codés en dur, exigences d'environnement, et un script check.sh/make check qui permet de recréer la DB en <5 min sur une machine propre.

_Quelques points appellent une attention particulière lors de la prochaine itération : ia_tool_declared._

## 3. Déclaration d'utilisation de l'IA

> La déclaration est très détaillée sur les étapes d'utilisation de l'IA, les validations humaines et les limites observées. En revanche, elle reste générique sur l'outil (nom sans version/modèle) — précise la version ou le modèle utilisé pour atteindre la note maximale.

**Sujets bien couverts dans votre déclaration :**

- à quelle étape l'IA a été utilisée
- comment la sortie a été validée par l'humain
- limites ou erreurs observées

**Sujets à ajouter ou expliciter pour la prochaine itération :**

- outils utilisés (nom + version/modèle)

## 4. Pistes d'action pour la prochaine itération

- Reprendre la requête de la section « Preuve » pour qu'elle s'exécute sur `db/nexamart.duckdb` et qu'elle produise la forme attendue (voir pistes en section 1).
- Réviser le brief en tenant compte des observations par dimension de la section 2.
- Compléter `ai-usage.md` en y ajoutant : outils utilisés (nom + version/modèle).

---

## 5. Traçabilité

- **Run ID :** `20260514T221333Z-7d34bf6a`
- **Devoir :** `S01`
- **Étudiant·e :** `jmpierre23`
- **Commit analysé :** `306dd0a`
- **Audit (côté instructeur) :** `tools/instructor/feedback_pipeline/audit/20260514T221333Z-7d34bf6a/jmpierre23/`
- **Prompts (SHA-256) :**
  - `sql_extractor_system` : `90ee9e277de7a27f...`
  - `rubric_grader_system` : `505f32d1d8319d66...`
  - `ai_usage_grader_system` : `81cb7fdf89bda55a...`
- **Fournisseur (rubrique) :** `openai`
- **Fournisseur (IA-usage) :** `openai` (gpt-5-mini-2025-08-07)

_Ce feedback a été produit par un pipeline automatisé et **revu par l'équipe pédagogique avant publication**. Aucun chiffre ni étiquette de niveau n'est diffusé à ce stade expérimental : l'objectif est uniquement formatif. Ouvrez une issue dans ce dépôt pour toute question._

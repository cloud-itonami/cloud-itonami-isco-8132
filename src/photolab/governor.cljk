(ns photolab.governor
  "PhotoLabCoordGovernor — the independent safety/scope layer gating
  every plant scheduling/logistics proposal an advisor may make for a
  photographic products (film/photo) processing plant crew. The
  governor never dispatches hardware itself, never operates
  film/photo-processing equipment (developer/fixer processing lines,
  rollers, processing machinery) itself, and never finalizes a
  processing-operation-execution decision or a chemical-safety-
  clearance decision, nor overrides a plant safety officer's judgment
  — those are permanently out of this actor's scope and remain a
  plant safety officer's exclusive judgment (README's 'Robotics
  premise': this actor coordinates PLANT SCHEDULING/LOGISTICS ONLY —
  it never operates processing equipment or makes chemical-safety-
  clearance decisions itself). Modeled on cloud-itonami-isco-8112's
  mineralplant.governor (closest domain shape — plant
  scheduling/logistics coordination for a hazardous-material
  industrial-site crew).

  HARD invariants (:hard? true, ALWAYS :hold, never overridable):
    1. operator provenance   — the photo-processing equipment
                                operator must be independently
                                verified/registered before any action.
    2. plant provenance      — the photo-processing plant site must be
                                independently verified/registered
                                before any action.
    3. no-actuation          — proposal :effect must be :propose (the
                                governor never dispatches hardware and
                                never operates processing equipment
                                itself; it only gates what the advisor
                                may coordinate).
    4. closed op-allowlist   — only :log-work-record,
                                :schedule-crew-operation,
                                :flag-safety-concern and
                                :coordinate-supply-order may ever be
                                proposed; anything else is refused.
    5. scope-excluded action — any proposal to directly finalize a
                                processing-operation-execution
                                decision, or a chemical-safety-
                                clearance decision, or to override a
                                plant safety officer's judgment, is a
                                hard, permanent block (checked both
                                against the proposed :op and, defense-
                                in-depth, against the proposal's
                                :rationale text — matched as full
                                finalization/execution ACTION phrases
                                such as \"finalize the processing-
                                operation decision\" / \"declare the
                                chemical safety cleared\" / \"override
                                the plant safety officer's judgment\",
                                never as bare nouns like \"developer\",
                                \"fixer\" or \"silver\", so the check
                                can never self-trip on the advisor's
                                own routine rationale text, e.g.
                                \"logged work record for operator …\"
                                or \"scheduled crew operation for
                                developer/fixer processing-line
                                shift …\" or \"…routed for plant safety
                                officer review\" — all three
                                legitimately contain bare nouns like
                                \"developer\", \"fixer\" and \"plant
                                safety officer\" but none is a
                                finalization action, and all are
                                exercised by
                                `governor-test/default-mock-advisor-proposals-never-self-trip-on-scope-exclusion`).
  ESCALATION invariants (:escalate? true, ALWAYS human sign-off
  regardless of confidence):
    6. :op :flag-safety-concern (a chemical-exposure / equipment-
                                condition concern always escalates to
                                a human, never auto-commits).
    7. :op :coordinate-supply-order above `supply-cost-threshold`.
    8. low confidence (< `confidence-floor`)."
  (:require [kotoba.lang.text :as str]
            [photolab.store :as store]))

(def confidence-floor 0.6)
(def supply-cost-threshold 2000)

(def allowed-ops
  #{:log-work-record :schedule-crew-operation
    :flag-safety-concern :coordinate-supply-order})

;; Defense-in-depth: none of these ops are ever in `allowed-ops`
;; above, so they are already refused by the closed-allowlist check
;; below; they are named again here — as explicit finalization/
;; execution ACTIONS, never bare nouns — so a future allowlist edit
;; cannot silently re-open this specific out-of-scope path without
;; also touching this list. Note this actor never operates
;; film/photo-processing equipment or makes chemical-safety-clearance
;; decisions itself — those stay a plant safety officer's exclusive
;; judgment.
(def ^:private scope-excluded-ops
  #{:finalize-processing-operation-decision :authorize-processing-operation
    :proceed-with-processing-operation :declare-chemical-safety-cleared
    :finalize-chemical-safety-clearance
    :override-plant-safety-officer-judgment
    :override-safety-officer-judgment})

;; Full finalization/execution ACTION phrases only — never bare nouns
;; ("developer", "fixer", "silver", "chromium", "roller", "chemical")
;; — so this can never match inside the mock advisor's own default
;; rationale text (which legitimately contains those bare nouns, e.g.
;; "developer/fixer processing-line shift" / "plant safety officer
;; review"). See
;; `governor-test/default-mock-advisor-proposals-never-self-trip-on-scope-exclusion`.
(def ^:private scope-excluded-phrases
  ["finalize the processing-operation decision" "finalize the processing operation decision"
   "proceed with the processing operation" "authorize the processing operation"
   "commence the processing operation"
   "declare the chemical safety cleared" "declare chemical safety cleared"
   "declare the chemical safety clearance" "finalize the chemical safety clearance"
   "override the plant safety officer's judgment"
   "override the safety officer's judgment"
   "override plant safety officer judgment"])

(defn- contains-excluded-phrase? [s]
  (let [s (str/lower (or s ""))]
    (boolean (some #(str/includes? s %) scope-excluded-phrases))))

(defn- hard-violations [proposal operator-record plant-record]
  (let [{:keys [op rationale]} proposal]
    (cond-> []
      (nil? operator-record)
      (conj {:rule :no-operator
             :detail "未登録 operator への提案は不可（operator record は独立して検証・登録済みでなければならない）"})

      (nil? plant-record)
      (conj {:rule :no-plant
             :detail "未登録 plant への提案は不可（plant record は独立して検証・登録済みでなければならない）"})

      (not= :propose (:effect proposal))
      (conj {:rule :no-actuation
             :detail "effect は :propose のみ許可（governor はプラント設備を直接操作しない）"})

      (not (contains? allowed-ops op))
      (conj {:rule :unknown-op
             :detail (str op " は closed op-allowlist に無い — 提案不可")})

      (or (contains? scope-excluded-ops op) (contains-excluded-phrase? rationale))
      (conj {:rule :scope-excluded-action
             :detail "現像処理オペレーション実行判断の確定・chemical safety clearance 判断の確定・plant safety officer の判断の上書きは、この actor の権限外 — 常に永続ブロック"}))))

(defn check
  "Assess a proposal against `request`/`context`/`proposal` and a
  `store` implementing `photolab.store/Store`. Pure — never mutates
  the store, never dispatches a processing operation."
  [request _context proposal store]
  (let [operator-record (store/operator store (:operator-id request))
        plant-record (some->> (:plant-id proposal) (store/plant store))
        hard (hard-violations proposal operator-record plant-record)
        hard? (boolean (seq hard))
        conf (or (:confidence proposal) 0.0)
        low? (< conf confidence-floor)
        supply-order-over-threshold?
        (and (= :coordinate-supply-order (:op proposal))
             (number? (:cost proposal))
             (> (:cost proposal) supply-cost-threshold))
        always-risky? (or (= :flag-safety-concern (:op proposal))
                           supply-order-over-threshold?)]
    {:ok? (and (not hard?) (not low?) (not always-risky?))
     :violations hard
     :confidence conf
     :hard? hard?
     :escalate? (and (not hard?) (or low? always-risky?))}))

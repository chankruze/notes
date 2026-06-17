  **What the system does today**

  **CandidateSelector** **(****app/services/assignment/candidate_selector.rb****)**
  - DEFAULT_LIMIT = 10 — caps candidates at 10
  - DEFAULT_RADIUS_KM = 25 — searches within 25km
  - Orders by distance ascending (nearest first)
  - Applies eligibility filters (active, available, has FCM device, onboarding complete)
  - For pandits additionally: checks package capability + no time conflict

  **AssignPartnersJob** **(****app/jobs/assign_partners_job.rb****)**
  - Calls CandidateSelector and iterates through **all** returned candidates
  - Creates an offer for each candidate simultaneously (line 15–24)
  - Dispatches push notification per offer

  **RespondToOfferService** **(****app/services/assignment/respond_to_offer_service.rb****)**
  - First partner to accept → job marked :assigned, all sibling :sent offers instantly expired (line 114, 163–174)
  - On reject → if no remaining :sent offers exist, job moves to :expired (line 128–133)


  **What needs to change / potential gaps**

  **1. If there are more than 10 eligible candidates nearby**
  
  DEFAULT_LIMIT = 10 hard-caps the selection (candidate_selector.rb:22). The 11th-nearest pandit or vendor never gets an offer. If the intent is to reach **all**

  available candidates within radius (not just top 10), the limit needs to be removed or raised significantly.

  **2. No retry / re-expansion when all candidates reject**
  
  In reject_offer! (respond_to_offer_service.rb:128–133): when the last :sent offer is rejected, the job transitions to :expired. There is no fallback — no

  re-queuing AssignPartnersJob with a wider radius or new candidates. The booking simply goes unassigned silently.

  **3.** **AssignPartnersJob** **has no idempotent re-run for newly available candidates**

  The job sends offers only to candidates who haven't received one yet (new_record? check at line 17). But if the job is re-enqueued after partial rejections, it

   will only pick up new candidates — this is fine by design but the re-enqueue never actually happens.

  **Recommended changes**

  **If you want to send to ALL available candidates (not capped at 10):**

  # candidate_selector.rb

  def call

    return PartnerAccount.none unless booking_address&.coordinates.present?

    partners_scope
      .where(
        "ST_DWithin(partner_addresses.coordinates, ?::geography, ?)",
        booking_address.coordinates,
        radius_km * 1000
      )
      .select("#{PartnerAccount.table_name}.*", "#{distance_sql} AS distance_meters")
      .order(Arel.sql("distance_meters ASC"))
      # Remove .limit(limit) or raise it significantly
  end


  **If you want a retry when all candidates reject (re-expand radius or re-enqueue):**

  In reject_offer! in respond_to_offer_service.rb, after expiring the job, re-enqueue AssignPartnersJob — but you'd need a way to track retry state (e.g., expand

   radius by 10km each round) to avoid infinite loops.


  **Bottom line**

  For the **exact scenario of 10 pandits + 10 vendors**, the current system already works correctly — all 10 of each get simultaneous offers and the first to accept

  wins. The risk area is if there are **more than 10** nearby (the 11th+ never gets notified), and there is **no retry** if all candidates reject.
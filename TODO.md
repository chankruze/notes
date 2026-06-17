- [ ] Assignment rules
	- [ ] Backend
	- [ ] MGMT Portal
- [ ] Manual partner assignment to booking
	- [x] Backend
	- [x] MGMT Portal
- [ ] Select options for customers, pandits, vendors
	- [x] Backend
- [ ] Backend
	- [ ] Export
		- [ ] Schema change to include related data
		- [ ] Column change
	- [x] Default sorting (last updated)
		- [x] Packages - Alphabetically
		- [x] Items - 
		- [x] Category - 
		- [x] Actors - (last updated)
	- [ ] Notifications
		
- [ ] Web - 25/march
	- [ ] Color palette
	- [x] Move status filter out
	- [ ] Column width according to the data length
	- [ ] Bulk action for all selected rows
	- [x] columns: -[email, dob, last updated] +[city, district, state]
	- [x] Paisa -> Rupee
	- [x] Minutes -> Hour (in UI)
	- [x] Send notification (announcement)
- [ ] Customer App
	- [ ] Search box text
	- [x] Pooja / Item delivery status
	- [ ] Address uniqueness
- [ ] Partner App
	- [ ] Play ring sound when offer screen is open also
	- [ ] When no actions, don't show the screen footer




```
account = PartnerAccount.where.not(fcm_token: [nil, ""]).last
# or:
# account = PartnerAccount.where.not(fcm_token: [nil, ""]).last

puts "Testing #{account.class} id=#{account.id} platform=#{account.fcm_platform}"

result = Fcm::Client.send_high_priority(
  token: account.fcm_token,
  title: "FCM Account Test",
  body: "Push test for account ##{account.id}",
  data: {
    type: "console_test",
    account_id: account.id,
    account_type: account.class.name,
    env: Rails.env
  }
)

puts({
  success: result.success?,
  message_id: result.message_id,
  error: result.error,
  http_status: result.http_status,
  response: result.response
}.pretty_inspect)

```

```ruby
bin/rails runner '                                 

partner = PartnerAccount.find_by!(phone: "7682036767")

booking = Booking.where(status: %w[paid confirmed]).order(created_at: :desc).first

job = booking.partner_assignment_jobs.find_or_initialize_by(partner_type: partner.partner_type)

job.status = :active                                                              

job.save!                                                                                      

offer = job.offers.create!(partner_account: partner, status: :sent, sent_at: Time.current)

Assignment::DispatchOfferJob.perform_now(offer.id)

puts({offer_id: offer.id, offer_uuid: offer.uuid, booking_id: booking.booking_id}.inspect)

'
```

```
bin/rails runner 'PartnerAssignmentJob.destroy_all'
```




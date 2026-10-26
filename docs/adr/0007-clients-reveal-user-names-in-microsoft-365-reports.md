# Clients turn off concealed names in Microsoft 365 reports before connecting

Microsoft 365 conceals user, group and site names in all usage reports by default, including the Microsoft 365 Copilot usage report: user principal names and display names arrive as hashes. With hashes we cannot match a Copilot Seat to a User, so an Idle Seat cannot be acted on. Turning the setting off is a precondition of the Microsoft 365 Copilot Connection: a Client admin turns off "Display concealed user, group, and site names in all reports" in the Microsoft 365 admin center. Token Management reads the setting through Microsoft Graph (`ReportSettings.Read.All`) on every Collection Run and puts the Connection into a needs-action state while names are concealed. It never changes the setting itself.

The setting applies to the whole tenant, so turning it off also reveals names in every other Microsoft 365 usage report to the Client's own report readers. The Client must accept that, and their works council may need to be told. Showing names is consistent with ADR 0003, which already shows Users by name in SAM-style facts.

## Considered options

- Reading Copilot interactions from the Purview audit log (Office 365 Management Activity API) and aggregating them as they arrive: rejected because it means handling per-call events, which ADR 0003 keeps out of the tool, and needs broader permissions and more processing.
- Leaving names concealed and showing only seat and activity counts: rejected because Idle Seats on Microsoft 365 Copilot, the most common Seat-licensed AI Tool at our Clients, would not be actionable.

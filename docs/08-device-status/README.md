# Device, network, and UI flow status

**Stage A** · the `08-device-status/` folder of the WhatsApp 2.26.35.75 (263507522) reverse engineering documentation.

device_ram, SIM fields, hasav, network_radio_type, cellular_strength, mistyped, method/reason, entrypoint, client_metrics, recaptcha, feo2_query_status, and other status parameters.

## Folder documents

| Document | Title |
|---|---|
| [cellular_strength.md](cellular_strength.md) | cellular_strength — cellular network signal level |
| [client_metrics.md](client_metrics.md) | client_metrics — request JSON metrics (attempts, install source, SIM indicators) |
| [device_ram.md](device_ram.md) | device_ram — device RAM size in gibibytes |
| [education-fields.md](education-fields.md) | education_screen_displayed, clicked_education_link, prefer_sms_over_flash — flash-call education state |
| [entrypoint.md](entrypoint.md) | entrypoint — where registration was launched from ("suma" \| "create_paa" \| absent) |
| [feo2_query_status.md](feo2_query_status.md) | feo2_query_status — FEO2 client capabilities query status ("client_capabilities_cached" \| "success_get_client_capabilities" \| …) |
| [hasav.md](hasav.md) | hasav — SMS automatic verification availability (Google Play Services SMS Retriever) |
| [hasinrc-db.md](hasinrc-db.md) | hasinrc, db — local registration state (rc2 storage) and the developer mode flag (db) |
| [method-reason.md](method-reason.md) | method, reason — OTP delivery method and code re-request reason |
| [mistyped.md](mistyped.md) | mistyped — code of the phone number entry method on the registration screen |
| [network_radio_type.md](network_radio_type.md) | network_radio_type — active network connection type (radio interface) |
| [recaptcha.md](recaptcha.md) | recaptcha — reCAPTCHA flow JSON state ({"stage":"ABPROP_DISABLED"}) |
| [roaming-airplane.md](roaming-airplane.md) | roaming_type, airplane_mode_type — roaming and airplane mode |
| [simnum-simtype-simstate.md](simnum-simtype-simstate.md) | simnum, sim_type, sim_state, read_phone_permission_granted — SIM card and phone permissions state |

Navigation for the entire guide: [docs/README.md](../README.md). Start: [Flow overview](../01-flow-overview.md).

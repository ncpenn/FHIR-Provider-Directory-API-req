# FHIR Plan-Net DB Mapping v26 — Stress Test Report

**Prepared for:** Stakeholder Review
**Date:** February 20, 2026
**Scope:** Complete review of mapping document against Da Vinci PDex Plan-Net IG v1.1.0, CMS-0057-F requirements, SQL Server implementation feasibility, and .NET/Firely SDK readiness.

---

## Executive Summary

The mapping document is **impressively thorough** for a v26 iteration — the Meta tab alone shows a disciplined decision log that most teams never produce. That said, a stress test is supposed to find cracks, so what follows is deliberately adversarial. I found **3 critical issues** that need resolution before stakeholders, **6 significant issues** that affect implementation but have known workarounds, and **~12 minor items** worth flagging for completeness.

The bottom line: this is a solid mapping that will hold up to scrutiny. The issues below are the kind that surface during implementation and testing — better to surface them now.

---

## Critical Issues (Must Resolve Before Presenting)

### CRIT-1: PractitionerRole ID format inconsistency between mapping sheets

The **PractitionerRole** mapping tab and Meta decisions define the resource ID as:

```
pr-{NETWORK_ID}-{PROVIDER_ID}-{LOCATION_ID}
```

But the **Req_SearchParameters** tab references a different format:

```
{PROVIDER_ID}-{LOCATION_ID}-{SPECIALTY_ID}
```

...and references `LOC_REP_GROUPS` as the backing table instead of `NETWORK_PARTICIPATION`.

These are two fundamentally different units of construction. The mapping tab treats a role as "this provider, at this location, in this network." The search params tab treats it as "this provider, at this location, for this specialty." These produce different resource counts and different cardinalities against the same provider. The search params tab also introduces `LOC_REP_GROUPS` as a core table, but the mapping tab never mentions it — the mapping tab is built entirely on `NETWORK_PARTICIPATION` and `NETWORK_REIMBURSEMENT`.

**Questions to resolve:**

- Which is the canonical unit of construction? If it's network-scoped (the mapping tab), then search by specialty requires joining `PROVIDER_SPECIALTIES` and returning multiple specialties *per* PractitionerRole. If it's specialty-scoped (the search tab), then each provider×location×specialty is its own role resource, which is more granular but creates many more resources.
- What is `LOC_REP_GROUPS`? The schema tab `Schema_LocRepGroups` has only 1 row with 2 columns. Is this a view? A junction table? It appears in the search params tab as a core backing table but is unexplained in the mapping.
- The product variant (`pr-{...}-prod-{PRODUCT_ID}`) adds a fourth dimension. How does that interact with whichever ID scheme is chosen?

**Recommendation:** Settle this before the stakeholder meeting. It affects resource count estimates, search implementation, and the HealthcareService/PractitionerRole cross-reference pattern.

---

### CRIT-2: Network profile canonical URL — the mapping may understate the profile requirement

The mapping treats Network as "Organization with type=ntwk" throughout. This is conceptually correct, but the Plan-Net IG defines **plannet-Network** as a *separate profile* with its own canonical:

```
http://hl7.org/fhir/us/davinci-pdex-plan-net/StructureDefinition/plannet-Network
```

This means your CapabilityStatement must declare *both* `plannet-Organization` and `plannet-Network` as supported profiles on the Organization resource type. Your Req_CapabilityStatement tab lists "Network" as a separate row (good), but the mapping tab's Network sheet says `Profile (Canonical) = plannet-Network` while the resource is still `Organization`. Make sure your Firely serialization sets `meta.profile` to the plannet-Network canonical on network resources, not plannet-Organization. The Inferno test suite will check this.

---

### CRIT-3: `_include` and `_revinclude` support not addressed

The Req_CapabilityStatement tab mentions `_include` support as SHOULD with a list of common includes (`PractitionerRole:practitioner`, `:organization`, `:location`, `:network`, `:endpoint`). But there is no implementation guidance anywhere in the document for how `_include` translates to SQL joins, nor is `_revinclude` mentioned at all.

For a provider directory API, `_include` is the primary mechanism consumers use to get a PractitionerRole *with* its Practitioner, Location, and Organization in a single round-trip. Without it, clients make N+1 requests. The Inferno Plan-Net test kit tests `_include` behavior.

**What's needed:** For each resource, document which `_include` and `_revinclude` parameters will be supported, and note the SQL join strategy (e.g., "PractitionerRole?_include=PractitionerRole:practitioner → JOIN PROVIDERS ON PROVIDER_ID, serialize both PractitionerRole + Practitioner into Bundle.entry[]").

---

## Significant Issues (Address During Implementation)

### SIG-1: Practitioner.telecom sourced from Location — architectural concern

The mapping puts telecom directly on `Practitioner` but sources it from `LOCATIONS` via `NETWORK_PARTICIPATION`. This means a Practitioner who works at 3 locations would have 3 phone numbers on the Practitioner resource (potentially duplicates if locations share phones, or confusingly different if they don't).

The Plan-Net IG *does* mark Practitioner.telecom as Must Support, so you can't simply omit it. But the more natural FHIR pattern is: Practitioner carries personal/direct contact (which you've correctly excluded), PractitionerRole carries role-specific contact (which you have), and Location carries facility contact (which you also have).

**Options:**
1. Populate Practitioner.telecom with the *primary* location's phone only (e.g., the location with the most recent NETWORK_PARTICIPATION effective date).
2. Omit Practitioner.telecom and accept that Must Support without data is permissible (MS means "populate if available" — you could argue the data lives on PractitionerRole/Location instead).
3. Populate with all location phones, deduplicated.

**Recommendation:** Option 1 or 2. Document the decision for auditors.

---

### SIG-2: InsurancePlan.network derivation via MARKET_ID is coarse-grained

The mapping derives InsurancePlan.network by chaining `PRODUCTS.INSURER_ID → INSURERS.MARKET_ID → NETWORKS.MARKET_ID`. This means *every* product under an insurer gets *every* network in that market. If an insurer has 5 networks and 10 products, all 10 products show all 5 networks, even if product A only participates in networks 1 and 2.

The mapping acknowledges this ("Pending: confirm whether a PRODUCT_NETWORKS table exists"), but for stakeholders, you should flag the impact: consumers searching `InsurancePlan?network={X}` may get false positives — products returned that don't actually use that network.

**Mitigation:** If no PRODUCT_NETWORKS junction exists, document this as a known approximation and consider a Phase 2 enhancement to add the junction table.

---

### SIG-3: ISO 639-2B → BCP-47 language crosswalk is a blocking dependency

Meta decision row 14 notes that `LANGUAGES.ISO_639_2B` (3-letter codes like "spa") must be converted to BCP-47 (2-letter codes like "es"). The mapping marks this as "crosswalk table required at implementation time."

This is a well-defined mapping (ISO 639-2B to 639-1 is a standard conversion), but it's a table that *must exist at deploy time*. If it's missing, every Practitioner.communication element will either be invalid (3-letter code in a BCP-47 field) or omitted.

**Recommendation:** Build the crosswalk table now. It's ~180 rows. Consider embedding it as a static C# dictionary rather than a DB table, since it never changes.

---

### SIG-4: HealthcareService.category crosswalk hardcodes SPECIALTY_IDs

The mapping hardcodes specific SPECIALTY_ID groupings for category derivation:
- Dental: IDs 1,2,6,7,8,11,12,17,18,20,21,23,32
- Vision: IDs 14,15,16,25,26,27,28,29,30,31
- Medical: IDs 19,22,24

But SPECIALTIES is explicitly tenant-scoped — different clients have different specialty rows. These hardcoded ID lists are from one client's QA environment. A vision-only client won't have IDs 1,2,6 at all, and may have IDs 14-16 with different semantics.

**Recommendation:** The category crosswalk should be configuration-driven per tenant, not hardcoded. Add a `CATEGORY_CODE` column to SPECIALTIES or maintain a per-tenant crosswalk table.

---

### SIG-5: Location.position (lat/long) marked as data quality gap with no remediation plan

The mapping notes GEO is nullable and says "Records with NULL GEO must be batch-geocoded." But there's no indication of how many locations have NULL GEO, or what the geocoding strategy is.

For a provider directory, geospatial search (`Location?near=lat|long|distance`) is a primary consumer use case. If a significant portion of locations lack coordinates, the directory is functionally incomplete for map-based search.

**Recommendation:** Run a count of NULL GEO records per tenant. If it's >5%, this needs an explicit remediation plan (batch geocoding service, address validation pipeline, etc.) before go-live.

---

### SIG-6: `near` search parameter not addressed

The Req_SearchParameters tab does not list `near` as a search parameter for Location, but this is one of the most common consumer queries. The Plan-Net IG CapabilityStatement includes `near` for Location. Implementing it in SQL Server requires spatial queries using the `geography` data type (which you already have in `LOCATIONS.GEO`):

```sql
WHERE GEO.STDistance(geography::Point(@lat, @long, 4326)) <= @distanceMeters
```

This is straightforward given your schema, but it needs a spatial index for performance. Flag it as a Phase 1 deliverable, not Phase 2.

---

## Minor Issues and Observations

### MIN-1: Organization.address search path unclear
The Req_SearchParameters tab says Organization address search is backed by `LOC_REP_GROUPS` with a note "Confirm exact join path from PAYEES to address." This is unresolved — PAYEES has its own ADDRESS1/ADDRESS2 columns (mapped in the Organization sheet), so the search should hit PAYEES directly, not LOC_REP_GROUPS.

### MIN-2: Practitioner search backed by CredentialingDb, not EnterpriseDb
The Req_SearchParameters tab says `Practitioner._id` and `Practitioner.identifier` are backed by `CredentialingDb.dbo.PROVIDERS`, but the Practitioner mapping sheet says `EnterpriseDb.dbo.PROVIDERS`. These may be the same table via cross-database view or synonym, but it should be explicit.

### MIN-3: `SPECIAL_NEEDS_PATIENTS_FLAG` → AccessibilityVS code
The Location mapping maps this flag to `adacomp`, but `SPECIAL_NEEDS_PATIENTS_FLAG` doesn't necessarily mean ADA-compliant. Consider `cultcomp` (culturally competent) or a display-only code. Validate the flag's actual business meaning.

### MIN-4: AFTER_HOUR_PHONE mapped as `use=temp`
The FHIR ContactPoint.use `temp` means "temporary" — not "after hours." Consider `use=work` with an extension or `availableTime` constraint, or just omit the `use` qualifier and add a `text` note.

### MIN-5: TTY phone mapping incomplete
Location telecom maps TTY_PHONE_NUMBER as `system=phone` with a note about `contactPointAvailableTime` for TTY. But the standard approach for TTY in FHIR is `system=other` with `extension:contactpoint-comment = "TTY"` or using a code from the ContactPointSystem value set extension. There is no TTY-specific system code in base FHIR.

### MIN-6: No `meta.profile` guidance in mapping
None of the resource mapping sheets mention setting `meta.profile` to the plannet canonical URL. The Firely SDK won't do this automatically — you need to set it on each resource. Inferno checks for it.

### MIN-7: Practitioner.active uses 3 flags; PractitionerRole.active uses 4 flags
Practitioner: `LIST_IN_DIRECTORY_FLAG AND NOT SUSPEND_PLANS_FLAG AND NOT VERIFICATION_EXPIRED_FLAG`. PractitionerRole: `NP.LIST_IN_DIRECTORY_FLAG AND NOT NP.SUSPEND_PLANS_FLAG AND PROVIDERS.LIST_IN_DIRECTORY_FLAG AND NOT PROVIDERS.SUSPEND_PLANS_FLAG`. The role check includes both participation-level AND provider-level flags (4 conditions), but the Practitioner check doesn't include any participation-level flags. This is probably intentional (Practitioner is person-level, role is participation-level), but confirm that a Practitioner won't show as `active=true` when all their roles are inactive.

### MIN-8: PractitionerRole.code crosswalk is thin
The crosswalk maps 6 PROVIDER_TYPE values to 3 FHIR codes (dentist, doctor, optometrist). The PractitionerRoleVS has ~20+ codes. If a future client has types like "hygienist", "nurse practitioner", or "physician assistant", the crosswalk will need expansion. Consider making it configurable.

### MIN-9: InsurancePlan.administeredBy always equals ownedBy
The mapping explicitly notes this and says "flag for review if TPA arrangements exist." For dental/vision plans, TPA arrangements are common (e.g., a dental plan administered by a third-party claims processor). If any current or near-term client uses a TPA, this will need a data source.

### MIN-10: Bundle serialization not discussed
The mapping covers individual resource construction but doesn't address Bundle assembly — pagination (`Bundle.link` with `next`), `Bundle.total`, `Bundle.type` (searchset), or `Bundle.entry[].fullUrl` format. These are required for search responses and Inferno tests them.

### MIN-11: `vread` interaction declared as SHOULD but no versioning scheme discussed
If `vread` (GET /{Resource}/{id}/_history/{vid}) is supported, you need a version tracking mechanism. The mapping uses `meta.lastUpdated` from source timestamps but doesn't discuss `meta.versionId`. If you don't implement vread, remove it from the CapabilityStatement.

### MIN-12: $export compliance still unresolved
The mapping correctly notes this is TBD. For stakeholders: CMS-0057-F requires $export for Patient Access and Payer-to-Payer APIs, but the requirement for Provider Directory $export is ambiguous in the final rule. The Plan-Net IG itself does NOT require $export. Recommend declaring it out of scope for Phase 1 with a clear rationale.

---

## .NET / Firely Implementation Notes

A few things the mapping doesn't address that will matter when you sit down to code:

**Firely SDK version:** The free `Hl7.Fhir.R4` NuGet package (as of early 2026) provides model classes and serialization. It does NOT provide a FHIR server framework. You'll build ASP.NET Core controllers that return `Hl7.Fhir.Model.Bundle` serialized via `FhirJsonSerializer`. Confirm you're using `Hl7.Fhir.R4` (not `Hl7.Fhir.R5`) since Plan-Net is R4.

**Profile validation:** The free Firely SDK includes a validator (`Hl7.Fhir.Validation`), but loading Plan-Net StructureDefinitions for validation requires either downloading the IG package or using Firely's `IResourceResolver`. Consider building a test harness that validates your output against the Plan-Net profiles before integration testing.

**Content negotiation:** The CapabilityStatement requires `application/fhir+json` (SHALL) and `application/fhir+xml` (SHOULD). Firely provides both `FhirJsonSerializer` and `FhirXmlSerializer`. Wire up ASP.NET content negotiation using `Accept` headers.

**CORS:** The CapabilityStatement correctly calls out `Access-Control-Allow-Origin: *` for the public directory. In ASP.NET Core, use `app.UseCors(policy => policy.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod())` — but scope it to the FHIR endpoints only, not your admin/management endpoints.

**SQL Server spatial index:** For the `near` search parameter, create a spatial index on `LOCATIONS.GEO`:
```sql
CREATE SPATIAL INDEX IX_LOCATIONS_GEO ON dbo.LOCATIONS(GEO)
USING GEOGRAPHY_AUTO_GRID;
```

---

## Summary Scorecard

| Resource | Elements Mapped | Elements GAP | Conformance Risk |
|----------|----------------|-------------|-----------------|
| Practitioner | 22 | 2 (usage-restriction, via-intermediary) | Low |
| Organization (PAYEE) | 10 | 4 (alias, partOf, contact, endpoint) | Low |
| Organization (INSURER) | 8 | 2 (partOf, endpoint) | Low |
| Network | 7 | 5 (telecom, address, coverage, endpoint, usage) | Medium — no coverage area |
| PractitionerRole | 14 | 3 (endpoint, usage, via-intermediary) | **High — ID format dispute** |
| Location | 14 | 3 (endpoint, usage, via-intermediary) | Low |
| InsurancePlan | 11 | 3 (coverageArea, plan.type, plan.network) | Medium — coarse network derivation |
| HealthcareService | 12 | 4 (delivery-method, coverageArea, usage, via) | Medium — hardcoded category crosswalk |
| OrgAffiliation | 11 | 3 (endpoint, telecom, usage) | Low |
| Endpoint | 0 | 9 (all) | Confirmed permanent GAP |

**Overall assessment:** The mapping covers ~109 element-level decisions across 9 resources with clear data lineage for each. The decision log in Meta is thorough and shows a mature requirements process. The three critical items (PractitionerRole ID inconsistency, Network profile canonical, and _include strategy) are all fixable before the stakeholder meeting. The GAPs are well-documented and defensible — particularly Endpoint, which is a data-availability constraint, not a design omission.

---

## Recommended Next Steps

1. **Resolve CRIT-1 immediately** — pick the PractitionerRole unit of construction and update both the mapping tab and search params tab to be consistent.
2. **Add a `meta.profile` row** to every resource mapping sheet.
3. **Add `_include` / `_revinclude` guidance** to the Req_SearchParameters tab.
4. **Build the BCP-47 crosswalk** and the HealthcareService category crosswalk as configuration tables.
5. **Run a NULL GEO count** per tenant and establish a geocoding remediation plan.
6. **Run Inferno Plan-Net test kit** against a mock server as early as possible — it will surface conformance issues that document review cannot.

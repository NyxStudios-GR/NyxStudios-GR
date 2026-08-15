# VORP Outbreaks

Persistent settlement-level outbreak, epidemic, contamination, medicine-shortage and quarantine simulation designed specifically to sit **on top of** the supplied `vorp_diseases` resource.

## Architecture

`vorp_diseases` remains the authority for an individual character's exposure rolls, incubation, stages, treatment, terminal deadlines and character disease persistence. `vorp_outbreaks` owns settlement statistics, environmental contamination, outbreak severity, medicine demand, quarantine, inter-town pressure and world consequences.

The integration uses the real `vorp_diseases` exports found in the supplied project: `ExposePlayer`, `GetDiseases` and `HasDisease`. Disease IDs are copied from its `shared/diseases.lua`, including `dysentery`, `cholera`, `typhoid`, `tuberculosis`, `influenza`, `rabies`, `sepsis` and the rest.

## Requirements

- RedM / Cfx.re with OneSync
- `oxmysql`
- `vorp_core`
- the supplied `vorp_diseases`
- `vorp_inventory` only when you use the default inventory adapter

Recommended `server.cfg` order:

```cfg
ensure oxmysql
ensure vorp_core
ensure vorp_inventory
ensure vorp_diseases
ensure vorp_outbreaks

add_ace group.admin vorp_outbreaks.admin allow
```

## Installation

1. Extract the folder as `vorp_outbreaks` inside your resources directory.
2. Import `sql/vorp_outbreaks.sql`.
3. Ensure dependencies in the order shown above.
4. Review every file under `config/`, especially town/source coordinates for your map setup.
5. Set `Config.Locale = 'el'` if you want Greek player-facing messages.
6. Leave `Config.Debug = false` in production.

The first boot creates cached default state for all configured settlements. Future state is persisted in SQL. On restart, timestamp deltas are used for bounded catch-up progression; `Config.OfflineProgressionMaxHours` prevents long downtime from unfairly destroying a town.

## Important model behavior

This is not a per-NPC disease simulator. Each settlement tracks statistical susceptible/infected/symptomatic/recovered/deceased values. Ambient NPC visuals are client-side representations only. Actual player infections are still decided by `vorp_diseases` through its server-side `ExposePlayer` export.

Natural severe epidemics are deliberately rare. The natural-emergence check only performs a final low-probability spark after sanitation/water/food prerequisite risks are already present. Admin `/outbreak start` exists for testing and authored events but is not how normal emergence works.

## Town configuration

Edit `config/towns.lua`:

```lua
my_town = {
  label='My Town',
  coords=vector3(0.0,0.0,0.0),
  radius=200.0,
  population=120,
  sanitation=60,
  waterCleanliness=70,
  foodCleanliness=65,
  medicalBase=8
}
```

The shipped towns are Valentine, Rhodes, Saint Denis, Blackwater, Strawberry, Armadillo, Tumbleweed, Annesburg, Van Horn and Emerald Ranch.

## Disease configuration

Edit `config/diseases.lua`. IDs must exist in `vorp_diseases`. `transmission` decides which world channels can contribute pressure; `baseCommunitySpread`, `recoveryDays`, `mortalityImpact`, `travelerSpread`, quarantine effectiveness and prerequisite risk values control settlement behavior.

Do not add a disease here unless `vorp_diseases` can accept the corresponding exposure kind. The outbreak resource never creates an independent character disease record.

## Contamination sources

Edit `config/contamination.lua`. Sources can represent wells, pumps or water points. Each persistent contamination tracks disease, strength, decay, discovery, closure and disinfection state.

Players can use:

```text
/drinkwell
```

near a configured water source. Drinking always looks ordinary. If the source is contaminated, the server passes an exposure into `vorp_diseases`; infection is not guaranteed.

Doctors can use:

```text
/outbreak cases [town]
/outbreak investigate [town]
/outbreak sample [sourceId]
/outbreaksample [sourceId]
```

`/outbreak investigate` works from accumulated case signals and does not instantly reveal hidden source state. Sampling produces confidence; only sufficiently strong evidence marks a source discovered.

Authorized staff can close/disinfect nearby sources with:

```text
/outbreakclose [sourceId]
/outbreakdisinfect [sourceId]
```

## Medicine

`config/medicine.lua` defines simulated local stocks and demand. Demand rises with active disease load. Stocks are consumed statistically over time and price multipliers move gradually toward shortage pressure.

External shops should query:

```lua
local multiplier = exports.vorp_outbreaks:GetMedicinePriceMultiplier('valentine', 'rehydration_tonic')
local stock = exports.vorp_outbreaks:GetMedicineStock('valentine', 'rehydration_tonic')
```

The resource intentionally does not hard-code your shop resource.

Register a supply delivery from a trusted server script:

```lua
exports.vorp_outbreaks:RegisterSupplyDelivery('valentine', 'rehydration_tonic', 20, source, {
  route='saint_denis_to_valentine'
})
```

or:

```lua
TriggerEvent('vorp_outbreaks:server:supplyDelivery', 'valentine', 'rehydration_tonic', 20, source, {})
```

## Inventory adapter

Default: `server/adapters/inventory.lua` uses `vorp_inventory` server exports. If your server uses another inventory, replace only that adapter while preserving:

```lua
InventoryAdapter.GetItemCount(source,item)
InventoryAdapter.HasItem(source,item,amount)
InventoryAdapter.RemoveItem(source,item,amount)
InventoryAdapter.AddItem(source,item,amount,metadata)
```

No core outbreak module needs to change.

## Economy integration

`server/adapters/economy.lua` emits:

```text
vorp_outbreaks:server:medicinePriceChanged
vorp_outbreaks:server:economyPressure
```

`economyPressure` contains shop activity, travel, production and delivery-reward multipliers. Your economy resource can listen and apply only the parts you want.

## Law and quarantine

Authorized jobs come from `Config.Authorities.Jobs`. Quarantine levels are 0 none, 1 advisory, 2 controlled, 3 restricted, 4 emergency.

Quarantine reduces statistical transmission only to the extent that compliance/enforcement supports it. Officer presence inside a quarantined settlement raises enforcement; it decays if unattended.

External law resources can listen for:

```text
vorp_outbreaks:server:quarantineChanged
vorp_outbreaks:server:quarantineViolation
```

The resource does not magically block town entry.

## Sanitation

Trusted server scripts can call:

```lua
exports.vorp_outbreaks:ImproveSanitation('valentine', 4.0, 'cleanup_job')
```

or emit `vorp_outbreaks:server:improveSanitation`. Sanitation directly affects environmental risk.

## Admin commands

ACE: `vorp_outbreaks.admin`

```text
/outbreak status [town]
/outbreak list
/outbreak start [town] [disease] [seedFraction]
/outbreak stop [town]
/outbreak contaminate [sourceId] [disease] [strength]
/outbreak sanitation [town] [0-100]
/outbreak medicine [town] [item] [amount]
/outbreak quarantine [town] [0-4]
/outbreak debug
/outbreak history [town]
```

Dangerous operations are validated server-side.

## Public UI

`/healthnotice` displays only immersive public information: broad severity, quarantine level and already-discovered unsafe sources. It never reveals hidden statistical infection values to civilians.

## Server exports

```lua
exports.vorp_outbreaks:GetTownHealth(town)
exports.vorp_outbreaks:GetOutbreak(town)
exports.vorp_outbreaks:GetSanitation(town)
exports.vorp_outbreaks:GetMedicineStock(town,item)
exports.vorp_outbreaks:GetMedicinePriceMultiplier(town,item)
exports.vorp_outbreaks:IsQuarantined(town)
exports.vorp_outbreaks:AddContamination(sourceId,disease,amount,cause,metadata)
exports.vorp_outbreaks:ReportExposure(source,disease,kind,intensity,context)
exports.vorp_outbreaks:AddMedicineStock(town,item,amount,reason)
exports.vorp_outbreaks:RegisterSupplyDelivery(town,item,amount,source,metadata)
exports.vorp_outbreaks:SetQuarantine(town,level,reason)
exports.vorp_outbreaks:ImproveSanitation(town,amount,reason)
```

Only call mutation exports from trusted server resources.

## Emitted server events

```text
vorp_outbreaks:server:outbreakStarted
vorp_outbreaks:server:outbreakDiscovered
vorp_outbreaks:server:outbreakResolved
vorp_outbreaks:server:severityChanged
vorp_outbreaks:server:contaminationCreated
vorp_outbreaks:server:sourceStatusChanged
vorp_outbreaks:server:sourceDisinfected
vorp_outbreaks:server:medicineShortage
vorp_outbreaks:server:medicinePriceChanged
vorp_outbreaks:server:supplyDelivered
vorp_outbreaks:server:quarantineChanged
vorp_outbreaks:server:quarantineViolation
vorp_outbreaks:server:economyPressure
```

A newspaper/telegram resource can subscribe to `severityChanged` and decide when to publish. The system intentionally does not announce a hidden Level 1 event globally.

## Logging

Important state changes are written to `outbreak_logs`. Set `Config.Logging.Webhook` for Discord webhook logging of major transitions. Do not put database credentials or server secrets in webhook payloads.

## Persistence and history

Main tables:

- `outbreak_town_state`
- `outbreak_disease_state`
- `outbreak_contamination`
- `outbreak_medicine_stock`
- `outbreaks`
- `outbreak_case_signals`
- `outbreak_logs`

Resolved outbreaks remain in `outbreaks` permanently with start/detection/resolution timestamps, peak simulated infected count, deaths, primary source and quarantine usage.

## Performance

The default simulation tick is five minutes. Player town checks are cheap coordinate comparisons, player exposure is interval-based, SQL writes are batched by persistence interval, NPC visuals only touch nearby ambient client peds, and the server never creates hundreds of simulated NPC disease entities.

For large servers increase `SimulationInterval`, `PlayerExposureInterval` and `PersistenceInterval` before reducing epidemiological complexity.

## Security

Client requests never decide infection, disease ID from hidden state, contamination strength, stock, prices, outbreak severity or quarantine. Source interactions are coordinate-validated server-side and rate-limited. Admin mutations require ACE. External mutation events are plain server handlers and should be invoked only by trusted server resources.

## Testing Valentine scenario

For a controlled test:

```text
/outbreak contaminate valentine_well_01 dysentery 0.71
```

Do not start the outbreak manually. Have players use `/drinkwell` near the configured source. Case signals and background environmental pressure will accumulate. Doctors can investigate/sample, authorities can close the source, and medicine demand will rise if the disease state grows. For faster staging tests only, temporarily reduce simulation intervals and/or use `/outbreak start valentine dysentery 0.03`.

## Troubleshooting

**No state loads:** confirm SQL import and `oxmysql` start order.

**Player infections never occur:** confirm `vorp_diseases` is started and that its `ExposePlayer` export is unchanged. Remember its default rare disease probabilities are intentionally very low.

**`/drinkwell` says no source nearby:** verify the configured source coordinates against your map/interior setup.

**Shop prices do not change:** your shop must call `GetMedicinePriceMultiplier`; this resource does not overwrite another resource's prices.

**Quarantine does not physically block roads:** expected. Connect a law/checkpoint resource to the quarantine events if you want barriers or penalties.

**Custom inventory:** replace only `server/adapters/inventory.lua`.

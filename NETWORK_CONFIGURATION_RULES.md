# Network Configuration Rules (LuCI)

Conventions used throughout this document:

- Effective requiredness in LuCI JS forms is mostly controlled by `rmempty`. In `form.js`, `rmempty = true` by default, so many fields that look important are still optional in the UI unless the code explicitly sets `rmempty = false`.
- Shared validators come from `modules/luci-base/htdocs/luci-static/resources/validation.js`. Important ones used here are:
  - `uinteger`: integer >= 0
  - `range(a,b)`: any decimal value between `a` and `b` inclusive
  - `max(n)`: any decimal value <= `n`; negative values also pass unless another validator blocks them
  - `ip4addr` / `ip6addr` / `ipaddr`: address or prefix unless `("nomask")` is used
  - `macaddr`: six colon-separated octets; the normal form enforces unicast MACs
  - `hexstring`: even number of hexadecimal characters
  - `string` with no parameter: effectively no content validation
- When LuCI uses the default validator stack, empty required fields fail with `Expecting: non-empty value`, while datatype failures fail with `Expecting: ...`.
- Primary source files for the rules below are:
  - `modules/luci-mod-network/htdocs/luci-static/resources/view/network/interfaces.js`
  - `modules/luci-mod-network/htdocs/luci-static/resources/view/network/dhcp.js`
  - `modules/luci-mod-network/htdocs/luci-static/resources/tools/network.js`
  - `modules/luci-base/htdocs/luci-static/resources/protocol/dhcp.js`
  - `modules/luci-base/htdocs/luci-static/resources/protocol/static.js`
  - `protocols/luci-proto-ppp/htdocs/luci-static/resources/protocol/pppoe.js`
  - `modules/luci-base/htdocs/luci-static/resources/validation.js`
  - `modules/luci-base/htdocs/luci-static/resources/form.js`
  - `modules/luci-base/htdocs/luci-static/resources/tools/widgets.js`

## 1. WAN Configuration

### Field: `name`
- **Type**: UCI identifier string
- **Required**: Yes when creating a new interface
- **Default**: None
- **Allowed Values**: `[A-Za-z0-9_]+`, maximum 15 characters, unique among `network` sections
- **Rules**:
  - This field is defined in the Add Interface modal in `interfaces.js`.
  - Validation is a combination of shared `uciname` validation and a custom uniqueness/length check.
- **Validation Logic**: `interfaces.js` sets `rmempty = false` and `datatype = 'uciname'`, then rejects any name already present in `uci.get('network', value)` and any name longer than 15 characters.
- **Error Conditions**:
  - Empty value -> `Expecting: non-empty value`
  - Contains characters outside `[A-Za-z0-9_]` -> `Expecting: valid UCI identifier`
  - Name already exists in `network` -> `The interface name is already used`
  - Length > 15 -> `The interface name is too long`

---

### Field: `proto`
- **Type**: enumerated string
- **Required**: Yes when creating a new interface
- **Default**: `none` in the modal widget
- **Allowed Values**: Any protocol returned by `network.getProtocols()`
- **Rules**:
  - The UI only offers registered LuCI protocol plugins.
  - A protocol may still reject creation through its `isCreateable()` hook.
- **Validation Logic**: `interfaces.js` builds the list dynamically from registered protocols. `network.js` defaults `isCreateable()` to success, but protocol-specific code may override it.
- **Error Conditions**:
  - Empty selection -> `Expecting: non-empty value`
  - Protocol plugin rejects creation -> `New interface for "<protocol>" can not be created: <reason>`

---

### Field: `device`
- **Type**: device or alias string selected through `widgets.DeviceSelect`
- **Required**: Yes for non-virtual interfaces in the Add Interface modal; required on the interface edit modal's `_net_device` widget
- **Default**: None
- **Allowed Values**: Existing runtime device names, alias names like `@<network>` when aliases are allowed, or custom strings accepted by `DeviceSelect`
- **Rules**:
  - `DeviceSelect` excludes `lo`.
  - In the interface editor, the device chooser allows bridges and excludes `@<current-interface>`.
  - No datatype regex is attached to this field.
- **Validation Logic**: `interfaces.js` sets `optional = false`; `tools/widgets.js` provides the filtered device inventory. Because no datatype is set, LuCI effectively enforces only "non-empty".
- **Error Conditions**:
  - Empty required value -> `Expecting: non-empty value`

---

### Field: `disabled`, `auto`
- **Type**: boolean flags
- **Required**: No
- **Default**: `auto` defaults to enabled in the edit modal; `disabled` has no explicit LuCI default
- **Allowed Values**: `0` or `1`
- **Rules**:
  - `disabled` marks the interface disabled.
  - `auto` controls whether the interface is brought up on boot.
- **Validation Logic**: Both are simple `form.Flag` widgets in `interfaces.js` with no custom validation.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `defaultroute`
- **Type**: boolean flag
- **Required**: No
- **Default**: Enabled
- **Allowed Values**: `0` or `1`
- **Rules**:
  - Applies on the interface Advanced tab.
  - Unchecking it suppresses installation of a default route for that interface.
- **Validation Logic**: `interfaces.js` defines it as a simple flag with `default = enabled`.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `peerdns`
- **Type**: boolean flag
- **Required**: No
- **Default**: Enabled when the protocol supports peer DNS
- **Allowed Values**: `0` or `1`
- **Rules**:
  - Only shown for protocols in `has_peerdns()`, including `dhcp` and `pppoe`.
  - When unchecked, the custom `dns` field is shown for those protocols.
- **Validation Logic**: `interfaces.js` gates the field through `has_peerdns(proto)`.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `dns`
- **Type**: dynamic list of IP strings
- **Required**: No
- **Default**: None
- **Allowed Values**:
  - WAN/custom DNS field: `ipaddr`
  - This accepts IPv4 or IPv6 addresses and, because `nomask` is not used, also accepts prefixes
- **Rules**:
  - For peer-DNS protocols the field is only active when `peerdns = 0`.
  - Multiple values are allowed.
- **Validation Logic**: `interfaces.js` assigns `datatype = 'ipaddr'`. In `validation.js`, `ipaddr` delegates to `ip4addr` or `ip6addr` without `nomask`, so both plain addresses and CIDR-like values pass.
- **Error Conditions**:
  - Invalid IPv4/IPv6 syntax -> `Expecting: valid IP address or prefix`

---

### Field: `dns_metric`, `metric`
- **Type**: unsigned integer
- **Required**: No
- **Default**: Placeholder `0`; no explicit saved default
- **Allowed Values**: integers >= 0
- **Rules**:
  - `dns_metric` controls DNS resolver ordering weight.
  - `metric` controls route metric ordering.
- **Validation Logic**: `interfaces.js` assigns `datatype = 'uinteger'`.
- **Error Conditions**:
  - Negative number, decimal, or non-numeric input -> `Expecting: positive integer value`

---

### Field: `multipath`
- **Type**: enumerated string
- **Required**: No
- **Default**: None explicitly set by LuCI
- **Allowed Values**: `off`, `on`, `master`, `backup`, `handover`
- **Rules**:
  - This is an Advanced tab choice list only; LuCI does not apply extra validation beyond the choice list.
- **Validation Logic**: `interfaces.js` creates a `RichListValue` with the five fixed choices.
- **Error Conditions**:
  - No extra LuCI-side validation beyond normal list selection behavior

---

### Field: `ip4table`, `ip6table`
- **Type**: free-form string or unsigned integer
- **Required**: No
- **Default**: None
- **Allowed Values**: any non-empty string, or any unsigned integer
- **Rules**:
  - The UI pre-populates known routing tables from the runtime table list.
  - Because the datatype is `or(uinteger, string)`, the `string` branch means any non-empty text is accepted.
- **Validation Logic**: `interfaces.js` sets `datatype = 'or(uinteger, string)'`. In `validation.js`, `string` with no parameter always succeeds.
- **Error Conditions**:
  - Empty value is allowed because the field is optional
  - In practice, there is no datatype-based rejection for non-empty input

---

### Field: `sourcefilter`
- **Type**: boolean flag
- **Required**: No
- **Default**: Enabled when shown
- **Allowed Values**: `0` or `1`
- **Rules**:
  - Only shown for protocols in `has_sourcefilter()`. Within this scope, PPPoE gets it; plain DHCP client does not.
- **Validation Logic**: `interfaces.js` gates the field by protocol name and assigns no custom validator.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `delegate`
- **Type**: boolean flag
- **Required**: No
- **Default**: Enabled
- **Allowed Values**: `0` or `1`
- **Rules**:
  - This Advanced tab field controls downstream IPv6 prefix delegation from the interface.
- **Validation Logic**: `interfaces.js` defines it as a simple flag with `default = enabled`.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `hostname`
- **Type**: string
- **Required**: No
- **Default**: Empty string, which means "send the device hostname"
- **Allowed Values**: empty string, `*`, or a shared `hostname` value
- **Rules**:
  - This is the DHCP client hostname field from `modules/luci-base/.../protocol/dhcp.js`.
  - `''` means "send the hostname of this device".
  - `'*'` means "do not send a hostname".
  - The field placeholder is loaded from `/proc/sys/kernel/hostname`.
- **Validation Logic**: `protocol/dhcp.js` sets `datatype = 'or(hostname, "*")'`. Shared `hostname` validation in `validation.js` accepts hostnames up to 253 chars, requires at least one non-numeric/non-dot character, and allows underscores.
- **Error Conditions**:
  - Invalid hostname syntax and not `*` -> `Expecting: One of the following: ...`

---

### Field: `broadcast`
- **Type**: boolean flag
- **Required**: No
- **Default**: Disabled
- **Allowed Values**: `0` or `1`
- **Rules**:
  - This is the DHCP client "use broadcast flag" option, not the static IPv4 broadcast field documented later.
- **Validation Logic**: `protocol/dhcp.js` defines a plain flag with `default = disabled`.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `clientid`
- **Type**: hexadecimal string
- **Required**: No
- **Default**: None
- **Allowed Values**: even number of hexadecimal characters
- **Rules**:
  - Used by the DHCP client as the client identifier.
- **Validation Logic**: `protocol/dhcp.js` sets `datatype = 'hexstring'`. In `validation.js`, `hexstring` requires pairs of hex digits.
- **Error Conditions**:
  - Odd-length or non-hex input -> `Expecting: hexadecimal encoded value`

---

### Field: `vendorid`
- **Type**: free-form string
- **Required**: No
- **Default**: None
- **Allowed Values**: any string accepted by the text box
- **Rules**:
  - LuCI does not attach a datatype validator to this field.
- **Validation Logic**: `protocol/dhcp.js` creates a plain `form.Value` without datatype or custom validate function.
- **Error Conditions**:
  - No LuCI-side content validation

---

## 2. LAN Configuration

Shared interface creation fields (`name`, `proto`, `device`, `disabled`, `auto`) use the same code path and validation rules described in WAN Configuration because `interfaces.js` drives both WAN and LAN interface editing.

---

### Field: `force_link`
- **Type**: boolean flag
- **Required**: No
- **Default**: `1` for static interfaces and `0` otherwise through the field's defaults map
- **Allowed Values**: `0` or `1`
- **Rules**:
  - Used on the interface Advanced tab.
  - The defaults map in `interfaces.js` auto-suggests `1` for `proto = static`.
- **Validation Logic**: No custom validation; only enumerated flag behavior.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `ip6assign`
- **Type**: numeric string
- **Required**: No
- **Default**: No explicit saved default; UI offers `""` (disabled) and `64`
- **Allowed Values**: any decimal value <= 128 according to LuCI's validator
- **Rules**:
  - Intended to represent an IPv6 delegated prefix length for the LAN/downstream interface.
  - Blank disables assignment.
  - LuCI does not enforce integer-only or non-negative input here.
- **Validation Logic**: `interfaces.js` sets `datatype = 'max(128)'`. In `validation.js`, `max()` uses decimal parsing only, so values like `64.5` or `-1` technically pass.
- **Error Conditions**:
  - Value > 128 -> `Expecting: value smaller or equal to 128`

---

### Field: `ip6hint`
- **Type**: hexadecimal string
- **Required**: No
- **Default**: Placeholder `0`
- **Allowed Values**: hexadecimal text with optional `0x` prefix; parsed value must be `< 0xffffffff`
- **Rules**:
  - Only active when `ip6assign` is between `33` and `64`.
  - Blank is allowed.
- **Validation Logic**: `interfaces.js` uses a custom validator: `/^(0x)?[0-9a-fA-F]+$/`, `parseInt(value, 16)`, and `n < 0xffffffff`.
- **Error Conditions**:
  - Non-hex syntax, parse failure, or parsed value >= `0xffffffff` -> `Expecting a hexadecimal assignment hint`

---

### Field: `ip6class`
- **Type**: list of strings
- **Required**: No
- **Default**: None
- **Allowed Values**: `local` plus dynamically discovered IPv6 prefix class names
- **Rules**:
  - The field is populated from runtime `ipv6-prefix` data on known networks.
  - LuCI does not apply a datatype validator to the list entries.
- **Validation Logic**: `interfaces.js` builds the choice list dynamically and leaves the field otherwise unconstrained.
- **Error Conditions**:
  - No LuCI-side content validation

---

### Field: `ip6ifaceid`
- **Type**: IPv6 host identifier string
- **Required**: No
- **Default**: Placeholder `::1`
- **Allowed Values**: `eui64`, `random`, or an IPv6 host ID whose upper 64 bits are zero (for example `::1` or `::1:2`)
- **Rules**:
  - Used when building a stable LAN IPv6 address from a delegated prefix.
- **Validation Logic**: `interfaces.js` sets `datatype = 'ip6hostid'`. In `validation.js`, this accepts `eui64`, `random`, or an IPv6 value with words 0-3 equal to zero.
- **Error Conditions**:
  - Any other input -> `Expecting: valid IPv6 host id`

---

### Field: `ip6weight`
- **Type**: unsigned integer
- **Required**: No
- **Default**: Placeholder `0`
- **Allowed Values**: integers >= 0
- **Rules**:
  - Higher values are preferred first during downstream prefix allocation.
- **Validation Logic**: `interfaces.js` uses shared `uinteger`.
- **Error Conditions**:
  - Negative number, decimal, or non-numeric input -> `Expecting: positive integer value`

---

### Field: `type`
- **Type**: enumerated string
- **Required**: Yes when creating a new device section
- **Default**: None
- **Allowed Values**: `""` (plain network device), `bridge`, `8021q`, `8021ad`, `macvlan`, `veth`, plus `bonding` or `vrf` only when system features are present
- **Rules**:
  - This is the device-level type field from `tools/network.js`.
  - For LAN scope, the relevant branch is `bridge`.
  - Changing to `bridge` updates downstream placeholders for dependent fields.
- **Validation Logic**: `tools/network.js` defines a fixed list of allowed types and updates dependent widgets in its `validate()` hook.
- **Error Conditions**:
  - Empty required value while creating a device section -> `Expecting: non-empty value`

---

### Field: `name_simple`, `name_complex`
- **Type**: device name string
- **Required**: Yes in device creation flows
- **Default**:
  - `name_simple`: current device name when editing
  - `name_complex`: current device name when editing
- **Allowed Values**:
  - `name_simple`: existing device chosen from runtime inventory
  - `name_complex`: any name with byte length <= 15 that is not already taken
- **Rules**:
  - `name_simple` is used for selecting an existing device to configure.
  - `name_complex` is used for creating a named logical device such as a bridge.
  - Existing device selection excludes duplicates and inactive Wi-Fi devices.
- **Validation Logic**:
  - `name_simple`: custom validator rejects device names that already have a `network.device` config section.
  - `name_complex`: `datatype = 'maxlength(15)'` plus custom rejection if the name already exists as another device section or as a live kernel device while creating.
- **Error Conditions**:
  - Existing device already has a config section -> `A configuration for the device "<name>" already exists`
  - `name_complex` longer than 15 bytes -> `Expecting: value with at most 15 characters`
  - `name_complex` already taken -> `The device name "<name>" is already taken`
  - Empty required value -> `Expecting: non-empty value`

---

### Field: `ifname_multi-bridge`
- **Type**: multi-select list of devices
- **Required**: No
- **Default**: Existing bridge ports when editing
- **Allowed Values**: runtime device names filtered through `DeviceSelect`
- **Rules**:
  - Used to attach wired ports to a bridge device.
  - Wi-Fi entries are only shown if they are already present.
  - Devices whose parent is the same bridge are filtered out.
- **Validation Logic**: `tools/network.js` uses `widgets.DeviceSelect` with `multiple = true` and no datatype validator; it relies on the filtered runtime choice list.
- **Error Conditions**:
  - No extra LuCI-side validation

---

### Field: `bridge_empty`
- **Type**: boolean flag
- **Required**: No
- **Default**: Disabled
- **Allowed Values**: `0` or `1`
- **Rules**:
  - Only shown when `type = bridge`.
- **Validation Logic**: Simple flag in `tools/network.js`.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `priority`, `ageing_time`
- **Type**:
  - `priority`: decimal range
  - `ageing_time`: unsigned integer
- **Required**: No
- **Default**:
  - `priority` placeholder `32767`
  - `ageing_time` placeholder `30`
- **Allowed Values**:
  - `priority`: any decimal 0..65535
  - `ageing_time`: integer >= 0
- **Rules**:
  - Both are only shown when `type = bridge`.
- **Validation Logic**: `tools/network.js` uses `range(0, 65535)` for `priority` and `uinteger` for `ageing_time`.
- **Error Conditions**:
  - `priority` outside range -> `Expecting: value between 0 and 65535`
  - `ageing_time` negative, decimal, or non-numeric -> `Expecting: positive integer value`

---

### Field: `stp`, `hello_time`, `forward_delay`, `max_age`
- **Type**:
  - `stp`: boolean flag
  - `hello_time`, `forward_delay`, `max_age`: decimal ranges
- **Required**: No
- **Default**:
  - `stp`: disabled
  - Timer fields: placeholders `2`, `15`, `20`
- **Allowed Values**:
  - `hello_time`: 1..10
  - `forward_delay`: 2..30
  - `max_age`: 6..40
- **Rules**:
  - Timer fields are only shown when `type = bridge` and `stp = 1`.
  - LuCI does not check cross-field relationships between the STP timers.
- **Validation Logic**: `tools/network.js` uses `range(1,10)`, `range(2,30)`, and `range(6,40)`.
- **Error Conditions**:
  - `hello_time` outside range -> `Expecting: value between 1 and 10`
  - `forward_delay` outside range -> `Expecting: value between 2 and 30`
  - `max_age` outside range -> `Expecting: value between 6 and 40`

---

### Field: `igmp_snooping`, `hash_max`, `multicast_querier`, `robustness`, `query_interval`, `query_response_interval`, `last_member_interval`
- **Type**:
  - `igmp_snooping`, `multicast_querier`: boolean flags
  - `hash_max`, `robustness`, `query_interval`, `query_response_interval`, `last_member_interval`: numeric
- **Required**: No
- **Default**:
  - `igmp_snooping`: disabled
  - `hash_max` placeholder `512`
  - `robustness` placeholder `2`
  - `query_interval` placeholder `12500`
  - `query_response_interval` placeholder `1000`
  - `last_member_interval` placeholder `100`
- **Allowed Values**:
  - `hash_max`, `query_interval`, `query_response_interval`, `last_member_interval`: unsigned integers
  - `robustness`: any decimal >= 1
- **Rules**:
  - `hash_max` depends on `igmp_snooping = 1`.
  - `robustness`, `query_interval`, `query_response_interval`, and `last_member_interval` depend on `multicast_querier = 1`.
  - `multicast_querier` has a defaults map that also toggles `igmp_snooping`.
- **Validation Logic**:
  - `hash_max`, `query_interval`, `last_member_interval`: shared `uinteger`
  - `robustness`: shared `min(1)`
  - `query_response_interval`: shared `uinteger` plus custom comparison requiring it to be lower than `query_interval`
- **Error Conditions**:
  - `hash_max`, `query_interval`, or `last_member_interval` negative, decimal, or non-numeric -> `Expecting: positive integer value`
  - `robustness` < 1 -> `Expecting: value greater or equal to 1`
  - `query_response_interval` not an unsigned integer -> `Expecting: positive integer value`
  - `query_response_interval >= query_interval` -> `The query response interval must be lower than the query interval value`

---

### Field: `mtu`, `macaddr`, `ipv6`, `mtu6`
- **Type**:
  - `mtu`: decimal range
  - `macaddr`: MAC string
  - `ipv6`: tri-state enable/disable/automatic
  - `mtu6`: decimal upper-bound only
- **Required**: No
- **Default**:
  - `mtu`: current device MTU used as placeholder
  - `macaddr`: current device MAC used as placeholder
  - `ipv6`: automatic based on sysfs
  - `mtu6`: no explicit default
- **Allowed Values**:
  - `mtu`: any decimal 576..9200
  - `macaddr`: six colon-separated hex octets, unicast
  - `ipv6`: automatic, enabled, or disabled
  - `mtu6`: any decimal <= 9200
- **Rules**:
  - `mtu` additionally must not exceed the parent device MTU for VLAN devices.
  - `mtu6` only appears when IPv6 is enabled.
  - `mtu6` uses `max(9200)`, so LuCI does not enforce integer-only or non-negative input.
- **Validation Logic**:
  - `tools/network.js` assigns `range(576, 9200)` to `mtu`, `macaddr` to `macaddr`, and `max(9200)` to `mtu6`.
  - The `mtu` field also runs a custom validator against the parent device MTU.
- **Error Conditions**:
  - `mtu` outside 576..9200 -> `Expecting: value between 576 and 9200`
  - `mtu` greater than parent VLAN MTU -> `The MTU must not exceed the parent device MTU of <n> bytes`
  - `macaddr` not a valid unicast MAC -> `Expecting: valid MAC address`
  - `mtu6` > 9200 -> `Expecting: value smaller or equal to 9200`

---

## 3. DHCP Configuration

This section combines three LuCI code paths:

- Per-interface DHCP/RA/DHCPv6 fields embedded in `view/network/interfaces.js`
- Global `dnsmasq` and `odhcpd` pages in `view/network/dhcp.js`
- Static lease and PXE/tag helper sections in `view/network/dhcp.js`

---

### Field: `dynamicdhcp`
- **Type**: boolean flag
- **Required**: No
- **Default**: Enabled
- **Allowed Values**: `0` or `1`
- **Rules**:
  - Only shown for DHCP server configuration attached to an interface with `proto = static`.
  - If disabled, only static leases are served.
- **Validation Logic**: Plain flag in `interfaces.js`.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `leasetime`
- **Type**: string
- **Required**: No
- **Default**:
  - Per-interface DHCP server: `12h`
  - Static lease override: no saved default; UI offers canned values
- **Allowed Values**:
  - Per-interface DHCP server: `infinite`, `deprecated`, or `/^[0-9]+[smhdw]?$/i`
  - Static lease override: any string; LuCI provides suggestions such as `5m`, `3h`, `12h`, `7d`, `infinite`
- **Rules**:
  - The per-interface field description says minimum `2m`, but the current LuCI validator only checks format and does not enforce the minimum.
  - The static lease field has no datatype or custom validator.
- **Validation Logic**:
  - `interfaces.js` uses a custom regex validator for the per-interface field.
  - `view/network/dhcp.js` leaves the host-specific field as a free-form `form.Value`.
- **Error Conditions**:
  - Per-interface field with bad format -> `Invalid DHCP lease time format. Use integer values optionally followed by s, m, h, d, or w.`
  - Static lease field -> no LuCI-side content validation

---

### Field: `force`, `dhcp_option`, `dhcp_option_force`
- **Type**:
  - `force`: boolean flag
  - `dhcp_option`, `dhcp_option_force`: dynamic lists of strings
- **Required**: No
- **Default**: None
- **Allowed Values**:
  - `force`: `0` or `1`
  - DHCP option lists: any string entries
- **Rules**:
  - These per-interface fields are only shown when the interface DHCP server is configured and `dnsmasq` support is present.
  - LuCI does not parse or validate DHCP option syntax.
- **Validation Logic**: `interfaces.js` uses plain flags and `DynamicList` widgets without datatypes.
- **Error Conditions**:
  - No LuCI-side syntax validation for option payloads

---

### Field: `dhcpv4`
- **Type**: enumerated string
- **Required**: No
- **Default**: None explicitly set by LuCI
- **Allowed Values**: `disabled`, `server`
- **Rules**:
  - Only shown for static interfaces when `dnsmasq` or `odhcpd` DHCPv4 support is available.
- **Validation Logic**: `interfaces.js` defines a fixed two-choice `RichListValue`.
- **Error Conditions**:
  - No extra LuCI-side validation beyond choice selection

---

### Field: `ipv6_only_preferred`
- **Type**: numeric string
- **Required**: No
- **Default**: `0`
- **Allowed Values**: exact literal `0`, or any decimal 300..86400
- **Rules**:
  - Only shown when `odhcpd` DHCPv4 support is available.
  - Because `range()` accepts decimals, LuCI does not enforce integer-only input for non-zero values.
- **Validation Logic**: `interfaces.js` sets `datatype = 'or(0, range(300,86400))'`.
- **Error Conditions**:
  - Any value other than literal `0` or a decimal in 300..86400 -> `Expecting: One of the following: ...`

---

### Field: `start`
- **Type**: numeric string or IPv4 address
- **Required**: No
- **Default**: `100`
- **Allowed Values**: unsigned integer offset, or IPv4 address without mask
- **Rules**:
  - Intended as the start of the DHCP pool.
  - LuCI accepts either an offset from the subnet base or a full IPv4 address.
  - LuCI does not verify that the chosen start fits inside the actual subnet.
- **Validation Logic**: `interfaces.js` sets `datatype = 'or(uinteger,ip4addr("nomask"))'`.
- **Error Conditions**:
  - Invalid input -> `Expecting: One of the following: ...`

---

### Field: `limit`
- **Type**: unsigned integer
- **Required**: No
- **Default**: `150`
- **Allowed Values**: integers >= 0
- **Rules**:
  - Represents the maximum number of leased addresses.
  - LuCI does not cross-check `limit` against subnet size or `start`.
- **Validation Logic**: `interfaces.js` sets `datatype = 'uinteger'`.
- **Error Conditions**:
  - Negative number, decimal, or non-numeric input -> `Expecting: positive integer value`

---

### Field: `netmask`
- **Type**: IPv4 string
- **Required**: No
- **Default**: None; placeholder is computed from the served interface subnet
- **Allowed Values**: any value accepted by shared `ip4addr`
- **Rules**:
  - This is the per-interface DHCP server "override netmask" field, not the static protocol netmask field.
  - LuCI intends it to be a netmask, but the datatype is `ip4addr`, so plain IPv4 addresses and prefix-style forms are also accepted.
- **Validation Logic**: `interfaces.js` sets `datatype = 'ip4addr'` and updates the placeholder from the interface's current subnet mask.
- **Error Conditions**:
  - Invalid IPv4 or prefix syntax -> `Expecting: valid IPv4 address or network`

---

### Field: `master`
- **Type**: boolean flag
- **Required**: No
- **Default**: None
- **Allowed Values**: `0` or `1`
- **Rules**:
  - Only one DHCP section across the whole config may be marked as designated master.
  - If another DHCP section already has `master = 1`, this field becomes read-only.
  - Toggling this field dynamically changes what RA/DHCPv6/NDP modes are selectable.
- **Validation Logic**:
  - `interfaces.js` scans all `uci.sections('dhcp', 'dhcp')` to find another master.
  - In the custom validator, RA server mode is forcibly disabled for protocols `dhcp`, `dhcpv6`, `3g`, `l2tp`, `ppp`, `pppoa`, `pppoe`, `pptp`, `pppossh`, `ipip`, `gre`, and `grev6`.
  - When `master = 1`, LuCI makes `server` unselectable for both `ra` and `dhcpv6` and may auto-switch existing `server` selections to `hybrid`.
- **Error Conditions**:
  - No popup validation error is thrown; LuCI enforces the rule by making the control read-only or unselectable

---

### Field: `ra`
- **Type**: enumerated string
- **Required**: No
- **Default**: None
- **Allowed Values**: `""` (disabled), `server`, `relay`, `hybrid`
- **Rules**:
  - `server` may be suppressed automatically depending on `master` and the interface protocol.
  - `relay` forwards RAs from the designated master interface.
- **Validation Logic**: `interfaces.js` defines the choice list, then mutates choice availability in the `master` validator.
- **Error Conditions**:
  - No explicit error text; invalid combinations are prevented by disabling the `server` choice

---

### Field: `ra_default`, `ra_slaac`, `ra_preference`, `ra_flags`, `_ra_pio_flags`
- **Type**:
  - `ra_default`, `ra_preference`: enumerated strings
  - `ra_slaac`: boolean flag
  - `ra_flags`, `_ra_pio_flags`: multi-select enumerations
- **Required**: No
- **Default**:
  - `ra_preference`: `medium`
  - `ra_flags`: UI defaults to `other-config` when unset
  - `_ra_pio_flags`: none
- **Allowed Values**:
  - `ra_default`: `""`, `1`, `2`
  - `ra_preference`: `low`, `medium`, `high`
  - `ra_flags`: `managed-config`, `other-config`, `home-agent`
  - `_ra_pio_flags`: `pd`
- **Rules**:
  - `ra_default` and `ra_slaac` depend on `ra = server` or `ra = hybrid` with `master = 0`.
  - `ra_flags` also depends on `ra = server` or `hybrid/master=0`.
  - `_ra_pio_flags` writes `dhcpv6_pd_preferred = 1` when `pd` is selected and unsets it when removed.
  - `ra_flags` removal writes `['none']` when the field is active and cleared.
- **Validation Logic**: All five are fixed-choice widgets in `interfaces.js`; there is no free-form text validation.
- **Error Conditions**:
  - No LuCI-side datatype errors; invalid combinations are handled through dependency visibility and write/remove hooks

---

### Field: `ra_pref64`
- **Type**: IPv6 CIDR
- **Required**: No
- **Default**: Placeholder `64:ff9b::/96`
- **Allowed Values**: valid IPv6 CIDR
- **Rules**:
  - Only shown when `ra = server` or `ra = hybrid` with `master = 0`.
- **Validation Logic**: `interfaces.js` sets `datatype = 'cidr6'`.
- **Error Conditions**:
  - Invalid IPv6 CIDR -> `Expecting: valid IPv6 CIDR`

---

### Field: `ra_maxinterval`, `ra_mininterval`, `ra_reachabletime`, `ra_retranstime`, `ra_lifetime`, `ra_mtu`, `ra_hoplimit`
- **Type**:
  - `ra_maxinterval`, `ra_mininterval`: unsigned integers
  - `ra_reachabletime`: decimal range 0..3600000
  - `ra_retranstime`: decimal range 0..60000
  - `ra_lifetime`: decimal range 0..9000
  - `ra_mtu`: decimal range 1280..65535
  - `ra_hoplimit`: decimal range 0..255
- **Required**: No
- **Default**: No explicit saved defaults; placeholders come from code or runtime
- **Allowed Values**:
  - `ra_maxinterval`, `ra_mininterval`: integers >= 0
  - `ra_reachabletime`: any decimal 0..3600000
  - `ra_retranstime`: any decimal 0..60000
  - `ra_lifetime`: any decimal 0..9000
  - `ra_mtu`: any decimal 1280..65535
  - `ra_hoplimit`: any decimal 0..255
- **Rules**:
  - All depend on `ra = server` or `ra = hybrid` with `master = 0`.
  - `ra_mtu` placeholder is loaded from `/proc/sys/net/ipv6/conf/<dev>/mtu`.
  - `ra_hoplimit` placeholder is loaded from `/proc/sys/net/ipv6/conf/<dev>/hop_limit`.
  - LuCI does not compare `ra_mininterval` against `ra_maxinterval`.
- **Validation Logic**: `interfaces.js` uses `uinteger` for the interval fields and `range()` for the bounded numeric fields.
- **Error Conditions**:
  - `ra_maxinterval` or `ra_mininterval` invalid -> `Expecting: positive integer value`
  - `ra_reachabletime` outside 0..3600000 -> `Expecting: value between 0 and 3600000`
  - `ra_retranstime` outside 0..60000 -> `Expecting: value between 0 and 60000`
  - `ra_lifetime` outside 0..9000 -> `Expecting: value between 0 and 9000`
  - `ra_mtu` outside 1280..65535 -> `Expecting: value between 1280 and 65535`
  - `ra_hoplimit` outside 0..255 -> `Expecting: value between 0 and 255`

---

### Field: `max_preferred_lifetime`, `max_valid_lifetime`
- **Type**: free-form string
- **Required**: No
- **Default**: None saved; placeholders `45m` and `90m`
- **Allowed Values**: any string; LuCI also offers canned values `5m`, `45m`/`90m`, `3h`, `12h`, `7d`
- **Rules**:
  - Both depend on `ra = server` or `ra = hybrid` with `master = 0`.
  - LuCI does not validate the duration syntax here.
- **Validation Logic**: `interfaces.js` uses `form.Value` with preloaded choices but no datatype.
- **Error Conditions**:
  - No LuCI-side syntax validation

---

### Field: `dhcpv6`, `dhcpv6_pd`, `dhcpv6_pd_min_len`
- **Type**:
  - `dhcpv6`: enumerated string
  - `dhcpv6_pd`: boolean flag
  - `dhcpv6_pd_min_len`: decimal range
- **Required**: No
- **Default**: None
- **Allowed Values**:
  - `dhcpv6`: `""`, `server`, `relay`, `hybrid`
  - `dhcpv6_pd`: `0` or `1`
  - `dhcpv6_pd_min_len`: any decimal 1..64
- **Rules**:
  - `dhcpv6_pd` only appears when `dhcpv6 = server`.
  - `dhcpv6_pd_min_len` only appears when `dhcpv6 = server` and `dhcpv6_pd = 1`.
  - As with `ra`, `master` may make `server` unselectable.
- **Validation Logic**: `interfaces.js` uses fixed choice widgets for `dhcpv6` and a `range(1,64)` validator for the minimum length.
- **Error Conditions**:
  - `dhcpv6_pd_min_len` outside 1..64 -> `Expecting: value between 1 and 64`

---

### Field: `dns`, `dns_service`, `dnr`, `domain`, `ntp`
- **Type**:
  - `dns`: list of IP addresses
  - `dns_service`: boolean flag
  - `dnr`: list of strings
  - `domain`: list of hostnames
  - `ntp`: list of hosts/IPs
- **Required**: No
- **Default**:
  - `dns_service`: enabled when shown
  - Others: none
- **Allowed Values**:
  - `dns`: `ipaddr("nomask")`, so plain IPv4 or IPv6 addresses only
  - `dnr`: any string because datatype is `string`
  - `domain`: shared `hostname`
  - `ntp`: `host(0)`, so hostname or IPv4/IPv6 address without mask
- **Rules**:
  - `dns` is shown for RA or DHCPv6 server/hybrid modes.
  - `dns_service` is only shown when the `dns` list is empty.
  - `dnr` describes a specific RFC 9463 syntax, but LuCI does not actually validate that syntax.
  - `ntp` suggestions include system NTP servers and local interface addresses when the local NTP server is enabled.
- **Validation Logic**:
  - `interfaces.js` sets `datatype = 'ipaddr("nomask")'` for `dns`, `datatype = 'string'` for `dnr`, `datatype = 'hostname'` for `domain`, and `datatype = 'host(0)'` for `ntp`.
- **Error Conditions**:
  - `dns` invalid IP syntax -> `Expecting: valid IP address`
  - `domain` invalid hostname -> `Expecting: valid hostname`
  - `ntp` invalid host/IP syntax -> `Expecting: valid hostname or IP address`
  - `dnr` -> no LuCI-side syntax validation

---

### Field: `ndp`, `ndproxy_routing`, `ndproxy_slave`
- **Type**:
  - `ndp`: enumerated string
  - `ndproxy_routing`, `ndproxy_slave`: boolean flags
- **Required**: No
- **Default**:
  - `ndp`: none
  - `ndproxy_routing`: enabled when shown
- **Allowed Values**:
  - `ndp`: `""`, `relay`, `hybrid`
  - Flags: `0` or `1`
- **Rules**:
  - `ndproxy_routing` appears for `ndp = relay` or `hybrid`.
  - `ndproxy_slave` appears for `ndp = relay` or `hybrid` with `master = 0`.
- **Validation Logic**: Fixed choice widgets in `interfaces.js`.
- **Error Conditions**:
  - No LuCI-side validation errors beyond choice gating

---

### Field: `authoritative`, `sequential_ip`, `address_as_local`
- **Type**: boolean flags
- **Required**: No
- **Default**:
  - `authoritative`: no explicit LuCI default
  - `sequential_ip`: no explicit LuCI default
  - `address_as_local`: no explicit LuCI default
- **Allowed Values**: `0` or `1`
- **Rules**:
  - These are `dnsmasq` global instance settings in `view/network/dhcp.js`.
- **Validation Logic**: Plain flag widgets, no custom validators.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `domain`
- **Type**: free-form string
- **Required**: No
- **Default**: None
- **Allowed Values**: any string
- **Rules**:
  - This is the `dnsmasq` local domain field, not the DHCPv6 announced domain list documented above.
- **Validation Logic**: `view/network/dhcp.js` creates a plain `form.Value` with no datatype.
- **Error Conditions**:
  - No LuCI-side content validation

---

### Field: `dhcpleasemax`
- **Type**: unsigned integer
- **Required**: No
- **Default**: Placeholder `150`
- **Allowed Values**: integers >= 0
- **Rules**:
  - Maximum number of active DHCP leases for the dnsmasq instance.
- **Validation Logic**: `view/network/dhcp.js` sets `datatype = 'uinteger'`.
- **Error Conditions**:
  - Negative number, decimal, or non-numeric input -> `Expecting: positive integer value`

---

### Field: `nonwildcard`, `interface`, `listen_address`, `notinterface`
- **Type**:
  - `nonwildcard`: boolean flag
  - `interface`, `notinterface`: network selectors
  - `listen_address`: IP selector
- **Required**:
  - `nonwildcard`: effectively defaults to enabled, but not required
  - Selectors: No
- **Default**:
  - `nonwildcard`: enabled
  - Others: none
- **Allowed Values**:
  - `interface`, `notinterface`: existing network names (`uciname`), because `nocreate = true`
  - `listen_address`: existing runtime IP addresses from the device inventory
- **Rules**:
  - `listen_address` and relay `local_addr` use `widgets.IPSelect`, which is non-creatable and therefore limited to discovered addresses.
  - `interface` and `notinterface` use `widgets.NetworkSelect` with `nocreate = true`.
- **Validation Logic**:
  - `NetworkSelect` uses `uciname`/`list(uciname)`.
  - `IPSelect` uses `ipaddr`/`list(ipaddr)` but with `create = false`, so choices come only from discovered interface addresses.
- **Error Conditions**:
  - Invalid selected network identifier -> `Expecting: valid UCI identifier`
  - Invalid selected IP -> `Expecting: valid IP address or prefix`

---

### Field: `logdhcp`, `logfacility`, `quietdhcp`
- **Type**:
  - `logdhcp`, `quietdhcp`: boolean flags
  - `logfacility`: enumerated string
- **Required**: No
- **Default**: None explicitly set
- **Allowed Values**:
  - `logfacility`: `KERN`, `USER`, `MAIL`, `DAEMON`, `AUTH`, `LPR`, `NEWS`, `UUCP`, `CRON`, `LOCAL0`-`LOCAL7`, `-`
- **Rules**:
  - `quietdhcp` depends on `logdhcp = 0`.
- **Validation Logic**: Fixed-choice or flag widgets in `view/network/dhcp.js`.
- **Error Conditions**:
  - No extra LuCI-side validation

---

### Field: `readethers`, `leasefile`
- **Type**:
  - `readethers`: boolean flag
  - `leasefile`: free-form string/path
- **Required**: No
- **Default**: None
- **Allowed Values**:
  - `readethers`: `0` or `1`
  - `leasefile`: any string
- **Rules**:
  - These are dnsmasq-global file settings.
- **Validation Logic**: Plain flag/string fields with no custom validation.
- **Error Conditions**:
  - No LuCI-side content validation

---

### Field: `local_addr`, `server_addr`, `interface`
- **Type**:
  - `local_addr`: selected IP address
  - `server_addr`: string with optional `#port`
  - `interface`: network selector
- **Required**:
  - `local_addr`: Yes for a usable relay entry
  - `server_addr`: Yes
  - `interface`: No
- **Default**: None
- **Allowed Values**:
  - `local_addr`: discovered runtime IP address from `IPSelect`
  - `server_addr`: IPv4 or IPv6 literal, optionally followed by `#<decimal-port>`
  - `interface`: existing network name
- **Rules**:
  - `server_addr` must be the same address family as `local_addr`.
  - Hostnames are not accepted in `server_addr`; the custom validator uses `validation.parseIPv4()` / `parseIPv6()` only.
  - The optional `interface` field restricts which interface replies are accepted on.
- **Validation Logic**:
  - `view/network/dhcp.js` validates `server_addr` by splitting on `#`, checking a decimal port if present, then requiring both endpoints to parse as IPv4 or both as IPv6.
- **Error Conditions**:
  - Either endpoint missing -> `Both "Relay from" and "Relay to address" must be specified.`
  - Port part present but not decimal -> `Expected port number.`
  - Address families differ or either side is not a parseable IP literal -> `Address families of "Relay from" and "Relay to address" must match.`

---

### Field: `enable_tftp`, `tftp_root`, `dhcp_boot`
- **Type**:
  - `enable_tftp`: boolean flag
  - `tftp_root`, `dhcp_boot`: free-form strings
- **Required**: No
- **Default**:
  - `tftp_root` placeholder `/`
  - `dhcp_boot` placeholder `pxelinux.0`
- **Allowed Values**:
  - `enable_tftp`: `0` or `1`
  - Strings: any text
- **Rules**:
  - `tftp_root` and `dhcp_boot` are only shown when `enable_tftp = 1`.
  - LuCI does not validate path or filename syntax.
- **Validation Logic**: Plain flag/string fields in `view/network/dhcp.js`.
- **Error Conditions**:
  - No LuCI-side content validation

---

### Field: `filename`, `servername`, `serveraddress`, `dhcp_option`, `networkid`, `force`, `instance`
- **Type**:
  - `filename`, `servername`, `serveraddress`, `networkid`, `instance`: strings
  - `dhcp_option`: list of strings
  - `force`: boolean flag
- **Required**:
  - `filename`, `servername`, `serveraddress`: Yes
  - Others: No
- **Default**:
  - `filename` placeholder `pxelinux.0`
  - `servername` placeholder `myNAS`
  - `serveraddress` placeholder `192.168.1.2`
- **Allowed Values**:
  - `filename`, `servername`, `serveraddress`, `instance`: any non-empty string when required
  - `networkid`: any string
  - `dhcp_option`: any string list entries
- **Rules**:
  - These are PXE/TFTP BOOTP host records.
  - Despite the label, `serveraddress` is not validated as an IP address.
  - `instance` is intended to match a dnsmasq instance name; LuCI populates the dropdown from existing instances.
- **Validation Logic**:
  - Requiredness comes from `optional = false`.
  - There is no datatype on `filename`, `servername`, `serveraddress`, `networkid`, or `instance`.
- **Error Conditions**:
  - Empty `filename`, `servername`, or `serveraddress` -> `Expecting: non-empty value`
  - No further LuCI-side syntax validation

---

### Field: tag section name
- **Type**: UCI identifier string
- **Required**: Yes when creating a new dnsmasq `tag` section
- **Default**: None
- **Allowed Values**: `uciname`, unique across existing DHCP tag names, their `!` negated forms, service names, and device names
- **Rules**:
  - The add-row UI uses `ui.addValidator()` directly on the create input.
  - Names cannot collide with service names or device names discovered at runtime.
- **Validation Logic**: `view/network/dhcp.js` uses shared `uciname` validation and a custom uniqueness check over:
  - existing `uci.sections('dhcp', 'tag')`
  - the same names with leading `!`
  - runtime `services`
  - runtime `devices`
- **Error Conditions**:
  - Invalid UCI identifier -> `Expecting: valid UCI identifier`
  - Duplicate/reserved-collision name -> `Name already exists. Choose a unique name.`

---

### Field: `dhcp_option` in tag sections
- **Type**: list of strings
- **Required**: No
- **Default**: None
- **Allowed Values**: any string list entries
- **Rules**:
  - Used by `tag` sections to attach DHCP options to tags.
  - LuCI does not parse or validate option syntax.
- **Validation Logic**: Plain `DynamicList` without datatype.
- **Error Conditions**:
  - No LuCI-side syntax validation

---

### Field: `match`, `vendorclass`, `userclass`
- **Type**: strings
- **Required**:
  - `match`: Yes
  - `vendorclass`: Yes
  - `userclass`: Yes
- **Default**:
  - `match` placeholder `61,8c:80:90:01:02:03`
  - Others: none
- **Allowed Values**: any non-empty string
- **Rules**:
  - LuCI documents expected syntax for `match`, but does not validate it.
  - `vendorclass` and `userclass` are also free-form.
- **Validation Logic**: These fields are required only by `rmempty = false` / `optional = false`; there is no datatype.
- **Error Conditions**:
  - Empty required field -> `Expecting: non-empty value`

---

### Field: `networkid`, `tag`, `match_tag`
- **Type**: strings or string lists
- **Required**:
  - `networkid` is required in match/VC/UC sections
  - `tag` and `match_tag` on static leases are optional
- **Default**: None
- **Allowed Values**:
  - `networkid` and `tag`: any tag name except reserved tags
  - `match_tag`: any tag value including reserved `known`, `!known`, `known-othernet`
- **Rules**:
  - Reserved tags are rejected by `validateTags()` for writable tag fields.
  - Negated tags like `!<tag>` are offered in choice lists.
- **Validation Logic**:
  - `validateTags()` rejects exact matches for `known`, `!known`, and `known-othernet`.
  - `match_tag` intentionally does not apply `validateTags()` so those reserved tags can be used there.
- **Error Conditions**:
  - Attempt to set a reserved tag in a validated field -> `Reserved tag`

---

### Field: `maindhcp`, `leasefile`, `leasetrigger`, `hostsdir`, `piodir`, `loglevel`
- **Type**:
  - `maindhcp`: boolean flag
  - `leasefile`, `leasetrigger`, `hostsdir`, `piodir`: free-form strings
  - `loglevel`: enumerated string
- **Required**: No
- **Default**: None
- **Allowed Values**:
  - `loglevel`: `0` through `7`
  - String/path fields: any string
- **Rules**:
  - These are global `odhcpd` settings.
  - LuCI does not validate file existence or script path validity.
- **Validation Logic**: `view/network/dhcp.js` uses plain flag/string widgets and a fixed list for `loglevel`.
- **Error Conditions**:
  - No LuCI-side content validation beyond normal list selection for `loglevel`

---

### Field: `url`, `arch`
- **Type**:
  - `url`: string
  - `arch`: numeric string
- **Required**:
  - `url`: Yes
  - `arch`: No
- **Default**:
  - `url` placeholder `tftp://[fd11::1]/pxe.efi`
  - `arch` default `""`
- **Allowed Values**:
  - `url`: any non-empty string because datatype is `string`
  - `arch`: any decimal 0..65535
- **Rules**:
  - `url` is documented as an RFC 3986 URL, but LuCI does not validate URI syntax.
  - `arch` offers many predefined IANA values, but the validator accepts any numeric value in range.
  - There is a UI typo in the canned values: value `39` is used twice, so the canned `40` label is not actually backed by value `40`.
- **Validation Logic**:
  - `view/network/dhcp.js` sets `datatype = 'string'` for `url` and `datatype = 'range(0,65535)'` for `arch`.
- **Error Conditions**:
  - Empty `url` -> `Expecting: non-empty value`
  - `arch` outside 0..65535 -> `Expecting: value between 0 and 65535`

---

### Field: `name` on static leases
- **Type**: hostname string
- **Required**: No
- **Default**: None
- **Allowed Values**:
  - Up to 256 characters total
  - Optional leading `*` and/or leading dot are stripped for validation
  - Each resulting label must match `^[a-z0-9_](?:[a-z0-9-]{0,61}[a-z0-9])?$` case-insensitively
- **Rules**:
  - Writing a hostname also forces `dns = 1` on that host section.
  - Removing the hostname also unsets the companion `dns` option.
- **Validation Logic**: `view/network/dhcp.js` uses custom `validateHostname()`, which is stricter and different from shared `hostname`.
- **Error Conditions**:
  - Length > 256 or invalid label syntax -> `Expecting: valid hostname`

---

### Field: `mac`
- **Type**: list of MAC strings
- **Required**: No, but either hostname or MAC must exist when assigning a static IPv4 address
- **Default**: None
- **Allowed Values**: one or more MAC strings matching `^(([0-9a-f]{1,2}|\*)[:-]){5}([0-9a-f]{1,2}|\*)$` case-insensitively
- **Rules**:
  - Wildcards are allowed.
  - Single hex digits are normalized to two-digit uppercase form for display.
  - LuCI rejects a MAC that is already used by another `dhcp host` section anywhere in the config.
- **Validation Logic**:
  - `view/network/dhcp.js` first performs a global duplicate-MAC scan across all `uci.sections('dhcp', 'host')`.
  - It then applies `isValidMAC()`.
- **Error Conditions**:
  - Duplicate MAC in another static lease -> `The MAC address <mac> is already used by another static lease in the same DHCP pool`
  - Invalid syntax -> `Expecting a valid MAC address, optionally including wildcards; invalid MAC: <mac>`

---

### Field: `ip`
- **Type**: IPv4 string or literal `ignore`
- **Required**: No
- **Default**: None
- **Allowed Values**: IPv4 address without prefix, or `ignore`
- **Rules**:
  - If an IPv4 address is set, LuCI requires at least one of hostname or MAC to be specified.
  - `ignore` is explicitly allowed and means ignore DHCP requests from the host.
  - A concrete IPv4 address must be unique among all `dhcp host` sections.
  - A concrete IPv4 address must belong to at least one active DHCP pool discovered by `getDHCPPools()`.
  - `getDHCPPools()` only considers non-ignored DHCP sections and only the first IPv4 address on each attached interface.
- **Validation Logic**:
  - `view/network/dhcp.js` sets `datatype = 'or(ip4addr,"ignore")'`, then applies the custom hostname/MAC, uniqueness, and pool-membership checks.
- **Error Conditions**:
  - Invalid IPv4 syntax and not `ignore` -> `Expecting: One of the following: ...`
  - No hostname and no MAC when a concrete IP is set -> `One of hostname or MAC address must be specified!`
  - Duplicate IPv4 in another static lease -> `The IP address <ip> is already used by another static lease`
  - IPv4 outside every discovered DHCP pool -> `The IP address is outside of any DHCP pool address range`

---

### Field: `duid`
- **Type**: list of DHCPv6 DUID or `DUID%IAID` strings
- **Required**: No
- **Default**: None
- **Allowed Values**:
  - `DUID`: even number of hex chars, length 20..260
  - Optional `%IAID`: hex chars, length 1..8
  - At most one `%`
- **Rules**:
  - Multiple values are allowed.
- **Validation Logic**: `view/network/dhcp.js` uses custom `validateDUIDIAID()`.
- **Error Conditions**:
  - More than one `%` -> `Expecting: maximum one "%"`
  - Invalid DUID length/content -> `Expecting: DUID with an even number (20 to 260) of hexadecimal characters`
  - Invalid IAID length/content -> `Expecting: IAID of 1 to 8 hexadecimal characters`

---

### Field: `hostid`
- **Type**: hexadecimal string
- **Required**: No
- **Default**: None
- **Allowed Values**: empty, or an even-length hex string whose length is 2..16 characters
- **Rules**:
  - Intended as an IPv6 token up to 64 bits.
  - Empty is allowed because the field is optional.
- **Validation Logic**: `view/network/dhcp.js` sets `datatype = 'and(rangelength(0,16),hexstring)'`.
- **Error Conditions**:
  - Length > 16 -> `Expecting: value between 0 and 16 characters`
  - Odd-length or non-hex input -> `Expecting: hexadecimal encoded value`

---

### Field: `instance`, `broadcast`, `dns`
- **Type**:
  - `instance`: string
  - `broadcast`, `dns`: boolean flags
- **Required**: No
- **Default**: None
- **Allowed Values**:
  - `instance`: any string; the UI preloads existing dnsmasq instances
  - `broadcast`, `dns`: `0` or `1`
- **Rules**:
  - `instance` binds a static lease to a specific dnsmasq instance when set.
  - `dns` may also be auto-set when a hostname is written.
- **Validation Logic**: Plain string/flag widgets in `view/network/dhcp.js`.
- **Error Conditions**:
  - No LuCI-side content validation for `instance`

---

## 4. PPPoE Configuration

Shared interface-level fields such as `name`, `device`, `defaultroute`, `peerdns`, `dns`, `dns_metric`, `metric`, `sourcefilter`, and `delegate` follow the WAN rules above because PPPoE is edited through the same `interfaces.js` modal plus the PPPoE protocol plugin.

---

### Field: `username`, `password`
- **Type**: strings
- **Required**: No
- **Default**: None
- **Allowed Values**: any string
- **Rules**:
  - LuCI does not mark either credential field as required.
  - `password` is only masked in the UI; it is not content-validated.
- **Validation Logic**: `protocol/pppoe.js` creates plain `form.Value` widgets without datatypes.
- **Error Conditions**:
  - No LuCI-side content validation

---

### Field: `ac`
- **Type**: string
- **Required**: No
- **Default**: Placeholder `auto`
- **Allowed Values**: any string
- **Rules**:
  - Empty means autodetect access concentrator.
- **Validation Logic**: Plain `form.Value` in `protocol/pppoe.js`.
- **Error Conditions**:
  - No LuCI-side content validation

---

### Field: `ac_mac`
- **Type**: MAC string
- **Required**: No
- **Default**: Placeholder `auto`
- **Allowed Values**: six colon-separated hexadecimal octets, unicast only
- **Rules**:
  - Empty means autodetect AC MAC.
- **Validation Logic**: `protocol/pppoe.js` sets `datatype = 'macaddr'`.
- **Error Conditions**:
  - Invalid MAC syntax or multicast MAC -> `Expecting: valid MAC address`

---

### Field: `service`
- **Type**: string
- **Required**: No
- **Default**: Placeholder `auto`
- **Allowed Values**: any string
- **Rules**:
  - Empty means autodetect service name.
- **Validation Logic**: Plain `form.Value` in `protocol/pppoe.js`.
- **Error Conditions**:
  - No LuCI-side content validation

---

### Field: `ppp_ipv6`
- **Type**: enumerated string
- **Required**: No
- **Default**: `auto`
- **Allowed Values**: `auto`, `0`, `1`
- **Rules**:
  - Only shown when the system has IPv6 support.
  - `reqprefix` and `norelease` depend on `ppp_ipv6 = auto`.
- **Validation Logic**: Fixed list in `protocol/pppoe.js`.
- **Error Conditions**:
  - No extra LuCI-side validation beyond choice selection

---

### Field: `reqprefix`
- **Type**: string
- **Required**: No
- **Default**: None
- **Allowed Values**: any string
- **Rules**:
  - Only shown when `ppp_ipv6 = auto`.
  - The help text suggests a length hint like `56` or a full prefix like `2001:db8::/56`, but LuCI does not validate either form.
- **Validation Logic**: Plain `form.Value` in `protocol/pppoe.js` with dependency only.
- **Error Conditions**:
  - No LuCI-side content validation

---

### Field: `norelease`
- **Type**: boolean flag
- **Required**: No
- **Default**: `1`
- **Allowed Values**: `0` or `1`
- **Rules**:
  - Only shown when `ppp_ipv6 = auto`.
  - `rmempty = false` keeps an explicit value in UCI.
- **Validation Logic**: Plain flag in `protocol/pppoe.js`.
- **Error Conditions**:
  - No LuCI-side validation errors

---

### Field: `_keepalive_failure`, `_keepalive_interval`
- **Type**:
  - `_keepalive_failure`: unsigned integer
  - `_keepalive_interval`: unsigned integer with minimum 1
- **Required**: No
- **Default**:
  - `_keepalive_failure` placeholder `5`
  - `_keepalive_interval` placeholder `1`
- **Allowed Values**:
  - `_keepalive_failure`: integer >= 0
  - `_keepalive_interval`: integer >= 1
- **Rules**:
  - These are UI helper fields that write a single UCI option `keepalive`.
  - If interval is blank/invalid, LuCI writes interval `1`.
  - If failure is blank/invalid and interval is `1`, LuCI removes `keepalive`.
  - If failure is blank/invalid and interval is not `1`, LuCI writes `5 <interval>`.
- **Validation Logic**:
  - `protocol/pppoe.js` sets `_keepalive_failure` to `uinteger` and `_keepalive_interval` to `and(uinteger,min(1))`.
  - `write_keepalive()` combines both helper fields into the saved `keepalive` string.
- **Error Conditions**:
  - `_keepalive_failure` invalid -> `Expecting: positive integer value`
  - `_keepalive_interval` invalid or < 1 -> `Expecting: positive integer value` or `Expecting: value greater or equal to 1`

---

### Field: `host_uniq`
- **Type**: hexadecimal string
- **Required**: No
- **Default**: Placeholder `auto`
- **Allowed Values**: even number of hexadecimal characters
- **Rules**:
  - The field is intended for raw host-uniq bytes.
- **Validation Logic**: `protocol/pppoe.js` sets `datatype = 'hexstring'`.
- **Error Conditions**:
  - Odd-length or non-hex input -> `Expecting: hexadecimal encoded value`

---

### Field: `demand`
- **Type**: unsigned integer
- **Required**: No
- **Default**: Placeholder `0`
- **Allowed Values**: integers >= 0
- **Rules**:
  - `0` means persistent connection.
- **Validation Logic**: `protocol/pppoe.js` sets `datatype = 'uinteger'`.
- **Error Conditions**:
  - Negative number, decimal, or non-numeric input -> `Expecting: positive integer value`

---

### Field: `mtu`
- **Type**: numeric string
- **Required**: No
- **Default**: Placeholder current device MTU, or `1500`
- **Allowed Values**: any decimal <= 9200 according to LuCI's validator
- **Rules**:
  - LuCI uses `max(9200)` only, so negative or decimal values technically pass.
- **Validation Logic**: `protocol/pppoe.js` sets `datatype = 'max(9200)'`.
- **Error Conditions**:
  - Value > 9200 -> `Expecting: value smaller or equal to 9200`

---

## 5. Static IP Configuration

Shared interface-level fields such as `name`, `device`, `defaultroute`, `dns`, `metric`, and `delegate` use the WAN/LAN rules already documented because the static protocol is plugged into the same interface modal.

---

### Field: `ipaddr`
- **Type**: IPv4 string or list of IPv4 CIDR/network strings
- **Required**: No in LuCI, even though the protocol is "static"
- **Default**: None
- **Allowed Values**:
  - Single-value mode: plain IPv4 address without mask
  - CIDR-list mode: values accepted by `or(cidr4, ipmask4)`
- **Rules**:
  - The widget starts in single-value mode when the stored value is a plain IPv4 address.
  - If the stored value already contains a `/`, or the user clicks the switch button, the widget becomes a dynamic list.
  - LuCI does not require at least one IPv4 address for the static protocol.
- **Validation Logic**:
  - `protocol/static.js` uses `ip4addr("nomask")` in single-value mode.
  - In list mode it uses `or(cidr4,ipmask4)`, which accepts IPv4 CIDR, IPv4 address/netmask notation, or even plain IPv4 addresses through the `ipmask4` chain.
- **Error Conditions**:
  - Single mode invalid IPv4 -> `Expecting: valid IPv4 address`
  - List mode invalid entry -> `Expecting: One of the following: ...`

---

### Field: `netmask`
- **Type**: IPv4 string
- **Required**: No
- **Default**: None; UI offers `255.255.255.0`, `255.255.0.0`, `255.0.0.0`
- **Allowed Values**: any plain IPv4 dotted-quad string
- **Rules**:
  - This field is hidden completely when `ipaddr` is already in CIDR/list mode.
  - LuCI does not verify that the value is a contiguous subnet mask.
- **Validation Logic**: `protocol/static.js` sets `datatype = 'ip4addr("true")'`, which only checks plain IPv4 dotted-quad syntax.
- **Error Conditions**:
  - Invalid dotted-quad IPv4 -> `Expecting: valid IPv4 address`

---

### Field: `gateway`
- **Type**: IPv4 string
- **Required**: No
- **Default**: None; placeholder may be filled from the current WAN gateway if exactly one WAN exists
- **Allowed Values**: plain IPv4 address without mask
- **Rules**:
  - The value must not equal any configured local IPv4 address on the same static interface.
  - LuCI does not validate subnet reachability beyond that equality check.
- **Validation Logic**:
  - `protocol/static.js` sets `datatype = 'ip4addr("nomask")'`.
  - A custom validator compares the gateway value against each `ipaddr` entry stripped to its address part.
- **Error Conditions**:
  - Invalid IPv4 syntax -> `Expecting: valid IPv4 address`
  - Gateway exactly equals one of the local IPv4 addresses -> `The gateway address must not be a local IP address`

---

### Field: `broadcast`
- **Type**: IPv4 string
- **Required**: No
- **Default**: None; placeholder is auto-calculated from `ipaddr` and `netmask`
- **Allowed Values**: plain IPv4 address without mask
- **Rules**:
  - LuCI calculates and shows the expected broadcast address as a placeholder.
  - The custom `validateBroadcast()` hook only updates placeholders; it does not reject mismatched broadcast values.
- **Validation Logic**: `protocol/static.js` sets `datatype = 'ip4addr("nomask")'` and always returns `true` from the custom validator.
- **Error Conditions**:
  - Invalid IPv4 syntax -> `Expecting: valid IPv4 address`
  - Mismatched but syntactically valid broadcast -> no LuCI-side error

---

### Field: `ip6addr`
- **Type**: list of IPv6 strings
- **Required**: No
- **Default**: None
- **Allowed Values**: values accepted by shared `ip6addr`, meaning plain IPv6 addresses or IPv6 prefixes
- **Rules**:
  - Multiple values are allowed.
  - The placeholder is `Add IPv6 address...`.
  - LuCI does not force `nomask`, so prefix notation is accepted.
- **Validation Logic**: `protocol/static.js` sets `datatype = 'ip6addr'`.
- **Error Conditions**:
  - Invalid IPv6 syntax -> `Expecting: valid IPv6 address or prefix`

---

### Field: `ip6gw`
- **Type**: IPv6 string
- **Required**: No
- **Default**: None
- **Allowed Values**: plain IPv6 address without prefix
- **Rules**:
  - Intended as the IPv6 default gateway.
- **Validation Logic**: `protocol/static.js` sets `datatype = 'ip6addr("nomask")'`.
- **Error Conditions**:
  - Invalid IPv6 syntax -> `Expecting: valid IPv6 address`

---

### Field: `ip6prefix`
- **Type**: IPv6 string
- **Required**: No
- **Default**: None
- **Allowed Values**: IPv6 address or prefix accepted by shared `ip6addr`
- **Rules**:
  - Intended as the public routed prefix delegated to the device for downstream distribution.
- **Validation Logic**: `protocol/static.js` sets `datatype = 'ip6addr'`.
- **Error Conditions**:
  - Invalid IPv6 syntax -> `Expecting: valid IPv6 address or prefix`

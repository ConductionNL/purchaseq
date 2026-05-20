# Spec: Discount and Promotion Engine

**Change:** catalog-purchase-management-other-t3
**Capability:** discount-engine

---

## ADDED Requirements

### REQ-DIS-001: Rule-Based Pricing Engine

The system must evaluate `PricingRule` objects against order lines to apply discounts — supporting coupon codes, per-item discounts, percentage discounts, and volume-based pricing.

#### Scenario: Apply percentage discount via matching rule

- **GIVEN** an active `PricingRule` with `discountType: "percentage"` and conditions matching the order line category and quantity
- **WHEN** `PricingRuleService::applyRules()` is called with those order lines
- **THEN** the discount percentage is applied to the unit price of matching lines
- **AND** the resulting order lines contain an `appliedDiscounts` annotation listing the rule slug and discount amount
- **AND** the original price is preserved alongside the discounted price

#### Scenario: Apply fixed per-item discount

- **GIVEN** an active `PricingRule` with `discountType: "fixed"` and `discountValue: 3.00`
- **WHEN** the rule conditions match an order line
- **THEN** EUR 3.00 is subtracted from the unit price of each matching line item
- **AND** the total discount amount is recorded in the `appliedDiscounts` annotation

#### Scenario: Apply coupon code discount

- **GIVEN** a `PricingRule` with a condition `{ "field": "couponCode", "operator": "eq", "value": "WELKOM2026" }`
- **WHEN** a user enters coupon code "WELKOM2026" at checkout and `applyRules()` is called
- **THEN** the coupon rule is matched and the discount is applied
- **WHEN** a user enters an invalid or expired coupon code
- **THEN** no discount is applied and a validation message is shown: "Onbekende of verlopen kortingscode"

### REQ-DIS-002: Rule Priority Ordering with First-match or All-match Execution Modes

Rules must be evaluated in priority order, and the execution mode must control whether evaluation stops at the first match or continues to apply all matching rules.

#### Scenario: First-match mode stops after first matching rule

- **GIVEN** two active `PricingRule` objects both matching the same order line, with priorities 1 and 10, both with `executionMode: "first-match"`
- **WHEN** `PricingRuleService::applyRules()` evaluates the order line
- **THEN** only the rule with priority 1 (lower number = higher priority) is applied
- **AND** the rule with priority 10 is not applied
- **AND** the `appliedDiscounts` annotation contains exactly one entry

#### Scenario: All-match mode applies all matching rules

- **GIVEN** two active `PricingRule` objects both matching the same order line, both with `executionMode: "all-match"`
- **WHEN** `PricingRuleService::applyRules()` evaluates the order line
- **THEN** both rules are applied, with discounts accumulated additively
- **AND** the `appliedDiscounts` annotation lists both rules

#### Scenario: Mixed execution modes — first-match rule encountered

- **GIVEN** the rule set contains both `first-match` and `all-match` rules, sorted by priority
- **WHEN** a `first-match` rule matches an order line
- **THEN** evaluation stops for that order line after the `first-match` rule fires
- **AND** subsequent `all-match` rules are not applied to that line

#### Scenario: Rules sorted by priority before evaluation

- **GIVEN** `PricingRule` objects with priorities 5, 1, 20, 3 exist in arbitrary creation order
- **WHEN** `PricingRuleService::applyRules()` is called
- **THEN** rules are evaluated in order: priority 1, 3, 5, 20
- **AND** priority ordering is ascending (1 = highest priority, 99 = lowest)

### REQ-DIS-003: Dynamic Pricing with Negotiated Discounts

The system must support negotiated discounts per customer type or catalog — reflecting framework agreement pricing for government and MKB customers.

#### Scenario: Government customer gets framework agreement pricing

- **GIVEN** a `PricingRule` with condition `{ "field": "customerType", "operator": "eq", "value": "overheid" }` and a 5% discount
- **AND** the current organization is configured as `customerType: "overheid"` in the `Administration`
- **WHEN** `applyRules()` is called for that organization's order
- **THEN** the 5% framework discount is applied to all eligible catalog order lines

#### Scenario: Discount not applied to expired rule

- **GIVEN** a `PricingRule` with `validUntil` in the past
- **WHEN** `PricingRuleService::applyRules()` is called
- **THEN** the expired rule is excluded from evaluation
- **AND** no discount from that rule appears in `appliedDiscounts`

### REQ-DIS-004: Manage Pricing Rules via Admin UI

Administrators must be able to create, edit, activate, and deactivate `PricingRule` objects through the Shillinq settings interface.

#### Scenario: Create a new pricing rule

- **GIVEN** an admin navigates to Instellingen → Kortingsregels
- **WHEN** the admin clicks "Nieuwe regel" and fills in name, conditions, discount type/value, execution mode, priority, and validity dates
- **THEN** a new `PricingRule` object is created via `ObjectService.saveObject()`
- **AND** the rule is immediately active if `status: "actief"` and current date is within validity range

#### Scenario: Deactivate a pricing rule

- **GIVEN** an active `PricingRule` exists
- **WHEN** an admin sets `status` to `inactief` and saves
- **THEN** the rule is excluded from all future `applyRules()` evaluations
- **AND** the change is recorded in the audit trail via `AuditTrailService`

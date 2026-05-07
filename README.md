# Printing & Signage Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source Management Information System (MIS) for print shops and signage studios -- bringing AI-assisted estimation, real-time production scheduling, and end-to-end job tracking to an industry locked into expensive, closed-source platforms.

Print shops and signage studios juggle wildly variable jobs across substrates, machines, finishing processes, and delivery timelines. Existing MIS platforms are exclusively proprietary, carry opaque per-seat pricing, and leave critical workflows -- outsourcing tracking, mobile shop-floor interaction, and profitability prediction -- poorly served. This project provides an open-source alternative that unifies estimation, production scheduling, inventory, delivery, and invoicing in a single system purpose-built for commercial print and wide-format signage operations.

---

## Why Printing & Signage Management?

- **Every incumbent is proprietary and expensive.** PrintPLANR, EFI PrintSmith Vision, ePS PACE, Clarity Software, ShopVOX, and Ordant are all closed-source commercial SaaS products with undisclosed or high per-seat pricing. There is no open-source MIS for the printing industry.
- **Web2Print and MIS pricing diverge.** A recurring complaint across ePS PACE, PrintSmith Vision, and other platforms is that Web2Print storefront prices and MIS-calculated production costs fall out of sync, causing customer-facing errors and margin leakage.
- **Outsourcing and trade print tracking is missing.** Few platforms handle outsourced job components within the production timeline. Printavo explicitly lacks this capability, and PACE handles it clumsily.
- **Shop floor interaction is desktop-bound.** Most platforms require desktop or remote-desktop access for production staff. True mobile-native apps for scanning, job sign-off, and status updates are rare (Cyrious Control, for example, has no mobile app at all).
- **No platform predicts profitability before job acceptance.** Estimators rely on templates and intuition; no incumbent proactively flags underpriced jobs using historical actuals or detects anomalous quotes before they are sent.

---

## Key Features

### Job Estimation and Quoting

- Template-driven costing for common job types (brochures, banners, vehicle wraps, signage) with material, labour, and overhead components
- Wide-format estimating with square-metre, linear-metre, and panel-count pricing
- Quote-to-order workflow with online customer approval, digital sign-off, and automatic conversion to a production job
- Dynamic pricing that auto-recalculates margins when material costs change

### Production Scheduling and Job Tracking

- Drag-and-drop Gantt scheduling across presses, cutters, laminators, and wide-format plotters
- Real-time job status via barcode or QR scan at each production stage, visible to both floor staff and customer-facing teams
- Multi-machine, multi-shift scheduling with setup times, substrate changeovers, and curing delays
- Outsourced component tracking within the overall job timeline

### Customer Portal and Proofing

- Customer self-service portal for job submission, file upload, proof approval, order history, and invoice access
- Proof management with upload, versioning, revision history, and automated email chasing for approvals
- Secure role-based access for commercially sensitive artwork files

### Delivery and Installation

- Delivery calendar with vehicle and crew assignment
- Installation booking for multi-day signage projects with sub-contractor management
- Digital proof-of-delivery with photos and electronic signature, linked directly to invoice generation
- Customer notification at key milestones

### Inventory, Invoicing, and Reporting

- Substrate, ink, and consumable tracking with per-job usage deduction and low-stock alerts triggering purchase orders
- Invoice generation on job completion with QuickBooks Online and Xero sync
- Job margin analysis: estimate vs. actual cost, press utilisation, and salesperson performance reporting

---

## AI-Native Advantage

This project targets AI capabilities that no incumbent currently offers in production. Estimation from natural language or uploaded artwork allows the system to auto-suggest substrates, quantity pricing, and production routes from a job description alone. Anomaly detection flags quotes that deviate from historical actuals for similar job types before acceptance, preventing underpriced jobs from reaching production. Automated scheduling optimisation rebalances the Gantt schedule when a job is delayed or a machine goes down, with downstream impact alerts. Demand forecasting predicts consumable consumption from the forward order book, reducing stock-outs and waste.

---

## Tech Stack & Deployment

The system targets cloud-first deployment with self-hosted and hybrid options. Integration with press and finishing equipment uses the CIP4 JDF/JMF open standard, which is freely implementable and already supported across the EFI Fiery DFE ecosystem. File handling supports PDF, AI, and EPS artwork with prepress workflow integration (e.g. Enfocus Switch). Accounting sync covers QuickBooks Online and Xero via their public APIs. PDF/X and ICC colour profiles follow ISO-governed open standards. The Gantt scheduling frontend requires virtualised rendering and WebSocket updates to handle dozens of machines and hundreds of concurrent jobs.

---

## Market Context

The printing software market centres on Management Information Systems that unify estimation, job management, scheduling, and billing. Incumbent pricing is largely opaque -- Printavo starts at $109/month for two users and $244/month for five; ShopVOX ranges from $99/month + $19/user to $199/month + $39/user; enterprise platforms like ePS PACE and Avanti Slingshot do not publish pricing at all. Primary buyers are print shop owners, production managers, and estimators at small-to-medium commercial print and wide-format signage businesses.

---

## Project Status

> This project is in the **research and specification phase**.
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.

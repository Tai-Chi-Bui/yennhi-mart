# UX Validation

## Prototyping
- **Tool**: Figma for mobile-first wireframes and high-fidelity prototypes.
- **Focus**: Search, checkout, delivery slots, multilingual UX (English/Vietnamese).

## Feedback
- **Beta Testing**: 100–500 users via feature flags in staging.
- **Tools**: Google Forms for feedback, Hotjar (free tier) for heatmaps/recordings.
- **In-App**: Feedback button for post-launch comments.

## A/B Testing
- **Tool**: Firebase A/B Testing for UX variations (e.g., checkout design).
- **Metrics**: Conversion rate, bounce rate, time-to-complete (Prometheus/Grafana).

## Grocery-Specific
- **Checkout**: Single-page, guest option, ≤1s (Stripe, Redis caching).
- **Delivery Slots**: Calendar view, ≤500ms (Firestore, Redis).
- **Substitutions**: Clear opt-in/out toggles for out-of-stock items.
- **Perishables**: Highlight expiration dates in cart.
- **Multilingual**: Django i18n/NestJS `i18next` for English/Vietnamese.

## Best Practices
- Accessibility: WCAG 2.1 (axe tool).
- Performance: Lighthouse for ≤2s loads.
- Monitoring: Track abandonment, session duration in Prometheus/Grafana, Hotjar.

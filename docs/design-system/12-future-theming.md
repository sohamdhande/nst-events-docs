# Future Theming

## Multi-Campus Support
NST-Events is architected for future multi-campus deployments. 

## Strict Rule: Token-Driven Branding
Absolutely **no campus-specific colors or logic** may be hardcoded into the component library. 
* E.g., `text-blue-600` is banned. `text-primary-base` is required.

## Implementation Strategy
When launching Campus B, we simply provide a new CSS variable token map (e.g., updating `--color-primary-base` from Blue to Crimson). The entire platform—Dashboard and Mobile App—will instantly re-theme itself without a single line of component code changing.

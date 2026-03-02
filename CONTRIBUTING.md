# Contributing

Thanks for your interest in contributing to the Kafka Cluster Sizing Calculator!

## How to Contribute

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-change`)
3. Make your changes to `index.html` (the entire app lives in this single file)
4. Test by opening `index.html` in a browser and verifying calculations
5. Commit your changes (`git commit -m "Add feature X"`)
6. Push to your branch (`git push origin feature/my-change`)
7. Open a Pull Request

## Guidelines

- The project is intentionally a **single self-contained HTML file** — keep it that way
- All CSS is inline in `<style>`, all JS is inline in `<script>`
- No external dependencies beyond Google Fonts CDN
- Every input change must trigger `recalc()` for live updates
- Use the existing CSS custom properties (`var(--accent-blue)`, etc.) for colors
- Use the existing helper functions (`rr()` for result rows, `mc()` for metric cards, `fmtMBs()`/`fmtTB()` for formatting)
- If adding a new calculation, include a formula tooltip via `data-tip` attributes

## Reporting Issues

Open a GitHub issue with:
- What you expected vs. what happened
- Input values you used (if it's a calculation issue)
- Browser and OS

## License

By contributing, you agree that your contributions will be licensed under the [Apache License 2.0](LICENSE).

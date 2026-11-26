# Warp SaaS: test results

Before each release is tagged, the test suite runs on the combinations of PHP, Laravel and database that the changes since the previous release can affect: every combination when database code, dependencies or the package setup changed, and fewer when only front-end files or non-database code changed. Each page says what ran and why. A version is released only when everything that ran passed.

| Version | Tested | Combinations |
| --- | --- | --- |
| [0.2.0](0.2.0.md) | 2026-11-25 | PHP 8.3, 8.4, 8.5 × Laravel 12, 13 × 7 databases |
| [0.1.1](0.1.1.md) | 2026-11-23 | PHP 8.3, 8.4, 8.5 × Laravel 12, 13 × 7 databases |
| [0.1.0](0.1.0.md) | 2026-11-20 | PHP 8.3, 8.4, 8.5 × Laravel 12, 13 × 7 databases |

# JavaScript Algorithms and Data Structures - freeCodeCamp

> **Source:** https://www.freecodecamp.org/learn/javascript-algorithms-and-data-structures/
> **Status:** ✅ Complete

![](https://img.shields.io/badge/Code-Javascript-informational?style=flat&logo=javascript&logoColor=yellow&color=f0db4f)

## Palindrome checker
Returns `true` if the provided string is a palindrome. Otherwise, returns `false`.

A palindrome is a word or phrase that is spelled the same forwards and backwards, ignoring punctuation, capitalization, and spacing.

[Solution](./comprobadorPalindormo.js)

## Roman numeral converter
Convert the provided number to a Roman numeral.

| Roman | Arabic |
|---|---|
| M | 1000 |
| CM | 900 |
| D | 500 |
| CD | 400 |
| C | 100 |
| XC | 90 |
| L | 50 |
| XL | 40 |
| X | 10 |
| IX | 9 |
| V | 5 |
| IV | 4 |
| I | 1 |

[Solution](./conversorRomanos.js)

## Caesar cipher
One of the simplest ciphers — letter values are shifted by a given amount. Implements ROT13 (shift by 13). All letters uppercase; non-alphabetic characters pass through unchanged.

[Solution](./cifradoCesar.js)

## Phone number validator
Returns `true` if the string matches a valid US phone number. Accepted formats:

```
555-555-5555
(555)555-5555
(555) 555-5555
555 555 5555
5555555555
1 555 555 5555
```

[Solution](./validadorNumerosTelefonicos.js)

## Cash register
`checkCashRegister(price, cash, cid)` returns an object with `status` and `change`:

- `{ status: "INSUFFICIENT_FUNDS", change: [] }` — not enough change
- `{ status: "CLOSED", change: [...] }` — cash in drawer equals change due
- `{ status: "OPEN", change: [...] }` — change returned from highest to lowest denomination

[Solution](./cajaRegistradora.js)

# When a Value Object adds value

1. **Always valid.** The concept has a rule of its own (a checksum, a format, a range), and the constructor enforces it, so the object can't exist in an invalid state. Code that receives one relies on the rule without checking it again. This applies even when the VO has a single field.
   - `Iban` rejects `FR76 3000 6000 0112 3456 7890 188` (wrong checksum) and stores a valid IBAN without the display spaces. Every `Iban` in the system has passed the checksum.
   - `Percentage` rejects `120`. `DateRange` rejects an end date before its start date.
   - Generic input cleanup (trimming, empty strings) is not a reason for a VO: it belongs at the input boundary. Normalisation belongs in a VO only when it is part of the concept, like removing the display spaces of an IBAN.

   | Concept | Concept-specific rule | VO worth it? |
   |---|---|---|
   | `Iban` | country-specific length, mod-97 checksum | Yes |
   | `Email` | `local@domain` format | Yes, if code relies on it (sending, uniqueness) |
   | `Percentage` | 0–100 | Yes |
   | `PostalCode` | format depends on the country | Not alone: the rule needs the country, so it belongs to `Address` (or a single-country `PostalCode`) |
   | `City` | none beyond "non-empty, max length" | No: generic constraints on a string field |

   - **Quick test:** can you state a rule that is specific to this concept and that code downstream relies on? If the only rules are "non-empty, max N characters", a plain string with field validation is enough.
   - **Enforcement:** make the constructor private and expose static named constructors (`Iban::fromString()`). The class is then the only way to build one. A factory in another class can't make the constructor private, so going through it stays a convention.
   - **Limit:** an ORM that hydrates objects without calling the constructor doesn't re-validate values loaded from the database.

2. **Behaviour next to the concept.** Operations that belong to the concept are methods on the VO. They have no side effects and return a new value.
   - `Money::add()` adds 10 EUR + 5 EUR = 15 EUR, and throws on 10 EUR + 5 USD instead of returning a meaningless 15.
   - `Money::allocate(3)` on 100 EUR returns 33.34 / 33.33 / 33.33. The total stays exactly 100, where a naive `100 / 3` loses a cent.
   - `DateRange::overlaps(DateRange $other)` tells whether two periods share at least one day, so double bookings are caught in one place.
   - `Address::isInSameCountry(Address $other)` decides, for example, whether domestic or international shipping applies.
   - **Limit:** a VO method uses only its own data and the values passed to it. Behaviour that needs a repository, a service, the clock or another aggregate goes in a domain service. Converting `Money` from EUR to USD needs exchange rates, so a `CurrencyConverter` service takes a `Money` and returns a new one.

3. **Related values kept together.** Values that only make sense as a set form one concept, and travel as one argument:
   - amount and currency (`100` alone means nothing);
   - start and end of a period;
   - latitude and longitude.
   - **Rules across values:** when a rule needs two values, the VO holding both owns it. `Address` checks the postal code against its own country. A `PostalCode` that receives the country from outside wouldn't keep track of which rule it passed, and storing the country in it too would duplicate the one in `Address`.

4. **Explicit meaning in signatures.** Compare `transfer(Money $amount, AccountId $to)` with `transfer(int $amount, string $to)`. With primitives, `transfer($accountId, $amount)` or passing cents where units were expected compiles and runs. With VOs, static analysis rejects it before the tests run. This is the cure for "primitive obsession".
   - Typical catches: swapped arguments, cents vs units, net vs gross amounts, metres vs kilometres.
   - **Limit:** this is the weakest reason on its own. It rarely justifies a VO by itself, given the mapping cost (ORM types, serializers, API layer). It usually comes for free with reasons 1 or 2.

5. **Immutable.** A VO can be shared safely. Changing a value means replacing the VO, never modifying it.
   - The bug this prevents: two order lines share the same `Price` instance, a discount calls `$price->setAmount(80)` on one line, and the other line silently changes too. With an immutable VO, `$line->price = $price->withDiscount(20)` creates a new object, and the other line is untouched.

6. **No identity.** Two VOs are equal when their values are equal: `Money(10, EUR)` equals another `Money(10, EUR)` but not `Money(10, USD)`. A VO is replaced rather than updated, and it's stored inside its entity (an embeddable or a custom column type), not in its own table.
   - By contrast, a `Customer` is an entity: two customers named "John Smith" are still two different people, told apart by their id.

**Rule of thumb:** create a VO when the concept has rules of its own (reason 1) or behaviour of its own (reason 2). Reasons 3 to 6 follow from that. A short-lived input with neither, such as a raw query parameter, doesn't need one: the object that carries it can enforce the rule.
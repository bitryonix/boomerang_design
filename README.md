# Boomerang: Bitcoin Cold Storage With Built-in Coercion Resistance

### What is Boomerang?

Imagine you have a lot of bitcoins - maybe for a company or personal savings - and you're worried not just about hackers, but about real-world threats like someone kidnapping you or threatening you to hand over the money. Regular "cold storage" (like keeping your keys offline on a hardware wallet) is great against online thieves, but it doesn't help if an attacker forces you to sign a transaction right away. That's where Boomerang comes in: it's a system designed to make stealing bitcoins through force way harder and riskier for bad guys.

### Why Does Boomerang Exist?

- **The Problem**: Big bitcoin holders (like businesses with millions in bitcoin) often have a few key people who control the ultimate access. Hackers can't easily get in, but a **"wrench attack"** - basically, physically forcing those people to transfer the money - works because normal systems let you send funds quickly and reliably.
- **The Goal**: Boomerang flips the script. It turns the withdrawal process into something unpredictable in timing during a certain period, with built-in ways to securely signal duress. Attackers can't count on getting the money in a set timeframe (or without risks), so they're less likely to try. Importantly, you can always withdraw eventually if everything's legit. It's just not instant or predictable at first, and becomes straightforward after a set time.

It's like hiding your treasure in a vault that takes a random, unknown, but within a wide range (selected by you privately), amount of time to open, and you can secretly hit a panic button without the attacker detecting if you have done such thing during the process. After a milestone (like 2 years), the vault can be opened normally and deterministically.

### How Does It Work?

Boomerang isn't for everyday spending. It's for long-term storage where you rarely need to touch the funds. Here's the basics:

1. **Setup Basics**:
   - You split control among a small group (e.g., 5 trusted people or "peers").
   - Each person uses special hardware devices (like secure cards or apps) to hold parts of the keys.
   - You also connect to neutral "watchtowers" (services that help coordinate) and "search and rescue services" (that can alert help if things go wrong).
   - You provide encrypted personal info (like your location) to the rescue service, locked with a secret password only you know.
   - Each peer privately sets their own range for how many "steps" (rounds of checks) their part of the process might take; say, a minimum and maximum number, like 6 months to 1 year worth of steps (tied to Bitcoin's blockchain timing). The group doesn't share these ranges; each person's device (Boomlet) randomly picks a number within their own private range when needed.
   - The group agrees on a future "milestone block 1" (a specific point on the Bitcoin blockchain, e.g., in 2 years) after which withdrawals become fast and predictable; no more randomness.
   - Prior to the milestone, the withdrawal is non-deterministic and the duress protection is in place. After that milestone, it is just a plain normal spending condition in a taproot address.

2. **Withdrawing Money**:
   - To move bitcoins, everyone in the group has to approve through a back-and-forth process.
   - In the boomerang period (before the milestone mentioned earlier, like the first 2 years), it involves multiple rounds of checks (the "digging game"). The exact number of steps for each peer's part is randomly picked by their Boomlet within their own private min-max range. No one (not even the group) knows the others' ranges or picks upfront, creating overall uncertainty in total time - so neither do attackers. This makes it hard for bad guys to plan around the timing, giving potential victims time (e.g., months) to be rescued if coerced.
   - During approvals, you can secretly signal "I'm under duress" (e.g., by choosing certain options in a quiz-like interface on a device). This looks normal to attackers but triggers the rescue service.
   - If anything seems off (like missed duress checks or messed up responses), the process just... stops. No money moves. But if all's good, it completes - you just don't know exactly how long it'll take ahead of time.
   - After the milestone block (e.g., 2 years in), you enter a "normal era" where everything is quick and deterministic, like regular cold storage.

3. **Duress Protection**:
   - The "secret signal" is easy: You memorize a set of 5 countries during setup as the consent set. Later, the system shows lists of countries, and your choices either confirm "all good" by entering the consent set or scream "help!" by entering any other combination of countries without obvious signs or stopping the ceremony. You signal the duress but the attacker observe no change compared to the non-duress situation.
   - If duress is signaled, the rescue service gets your info and can start a "search and rescue" - like contacting authorities or trusted contacts.

The whole thing uses encryption and anonymous networks (like Tor) to keep communications hidden. It's not foolproof against everything, but it makes coercion a bad bet for attackers: too uncertain in duration and too risky.

### Who Is It For?

- **Big Holders**: Companies or wealthy individuals with bitcoin they don't need to access often (e.g., long-term reserves).
- **High-Risk Situations**: Places where physical threats are real, like in unstable regions.
- **Not For**: Casual users or quick trades; it's too slow and complicated for that.

In short, Boomerang is like a bitcoin safe with a randomized time-delay lock and a hidden alarm during the boomerang phase, switching to normal access after a milestone. It prioritizes ultimate protection over convenience, making it tougher for anyone to force you out of your bitcoin. If you're curious about setting it up or the costs, it involves hardware, fees for services, and coordinating with others.

### What have we done so far?

We have designed the protocol and the full message sequence diagrams for [setup](setup) and [withdrawal](withdrawal). We have also made a [proof-of-concept implementation](https://github.com/bitryonix/boomerang) in Rust.

If you like to see the SVG files of the message sequence diagrams, here they are for the [setup](setup/setup_diagram_without_states.svg), [initiator peer's withdrawal](withdrawal/initiator_withdrawal_diagram_without_states.svg) and [non-initiator peer's withdrawal](withdrawal/non_initiator_withdrawal_diagram_without_states.svg) ceremonies.

### More info

You can find a deep dive in our [DEEPDIVE.md](DEEPDIVE.md) file.

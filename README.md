# six-axis-flight

A flight model for a spaceship with six degrees of freedom: it pitches, rolls and yaws freely and thrusts along its nose, deck and wings (throttle, lift, strafe). Flight Assist gives three ways to fly: arcade handling in open space, a drone-style hover near a landing pad, and pure momentum with it off.

It is one plain Rust file with no dependencies, so it runs in Bevy, any other Rust engine, a test or a server. One tick function, `step6`, moves the ship; everything it needs comes in as plain structs.

**Not for:** collisions, physics bodies or networking. It moves one ship's position, velocity and attitude; your game does the rest.

## Quick start

```toml
[dependencies]
six_axis_flight = { git = "https://github.com/GraySun2/six-axis-flight" }
```

Or copy `src/lib.rs` into your game as `flight6.rs`. Then, once a fixed tick:

```rust
use six_axis_flight::*;

let (engine, tuning) = (Engine::default(), Tuning::default());
let mut fly = FlyState::new(0.0, &engine, &tuning);
let mut body = Body { pos: V3::ZERO, vel: V3::ZERO, fuel: 100.0 };

let controls = Controls { main: 1.0, yaw: 0.3, ..Default::default() };
let ctx = Ctx { engine, tuning, ..Ctx::open(1.0 / 60.0) }; // open space, no gravity
let out = step6(&mut fly, &mut body, &controls, &ctx);
// body.pos, body.vel and fly.att.quat() now hold the new state; out says what the engines did.
```

`examples/bevy-fly` is a small Bevy 0.20 game that flies it with the keyboard, including the change from the model's z-up axes to Bevy's y-up. Run it with `cargo run --release` from that folder (a long first build).

## Use it in a game

1. Add the crate, or copy `src/lib.rs` into the game as `src/flight6.rs` with `mod flight6;`.
2. Keep one `FlyState` and one `Body` per ship. `FlyState::new(heading, &engine, &tuning)` starts level, at rest, with a full charge and Flight Assist on.
3. Each fixed tick, fill a `Controls` from the player's input and call `step6(&mut fly, &mut body, &controls, &ctx)`. `Ctx::open(dt)` is open space with no gravity; set `gravity`, `field` (a `HoverField`) and `gear` for the game's world.
4. Copy `body.pos` and `fly.att.quat()` onto the ship's transform. In the model's own axes the ship's nose is local +x, its deck up +z and its left wing +y. The model is z-up; for a y-up engine such as Bevy, turn both with the change of axes in `examples/bevy-fly` (`to_bevy`), and build the ship mesh with its nose along +X and its deck up along +Y. A mesh facing Bevy's usual -Z would fly sideways.
5. Use the returned `FlyOut` for the HUD and sound: `main_frac`, `lift_frac` and `small_frac` for engine glow and volume, `boosting`, `jinking`, `hover_assist` and `grip_on` for indicators, `accel` for camera shake, `fuel_used` and `charge_used` for gauges.

To change the feel, build the context as `Ctx { engine, tuning, ..Ctx::open(dt) }`; `Ctx::open` alone uses the defaults. Turn Flight Assist on and off with `fly.fa`.

For a hover field near a pad, `field_fade(distance, full, zero)` gives a smooth fade; multiply the fade for height by the fade for distance from the pad's axis, and pass the field's full-strength pull as `accel` with that `fade`, and the height above the pad as `h`. The hover assist assumes a level pad with +z up. One prototype used a 5 m/s² pull, full strength up to 450 m above the pad and none from 600 m.

## What it gives you

- **Axes.** World z is up. Body rates are `w = V3 { x: roll, y: yaw, z: pitch }` in rad/s; positive is right wing down, nose right, nose up. The ship's own axes are `f` (nose), `u` (deck up) and `r = f × u` (right wing).
- **Flight Assist on, open space.** Main thrust along the nose, retro as a brake, lift and strafe on the small thrusters. The grip swings the velocity onto the nose (so the ship carves), drag slows it gently, a held bank yaws the ship toward the low wing, and the wings level themselves soon after the roll stick is let go. `no_grip` lets it slide, for example while aiming.
- **Flight Assist on, in a hover field.** The stick sets a speed over the ground (up to `hover_vh`), lift sets a climb or sink, and the ship leans into the push up to `hover_tilt`, holding still hands off. The hover assist is on above a field fade of `hover_fade`, at or under `hover_speed`, or at any speed with the gear down. Main thrust from `punch_min_h` up leaves the hover, with the gear up only. Under `soft_h` with the gear down, the descent is held to `max(soft_min, h / 5)`. With Flight Assist on, the gear down caps the speed at `hover_speed` everywhere, open space included.
- **Flight Assist off.** Every thruster pushes as it is. A spin keeps going when the stick is let go, decaying slowly; `kill_rot` stops it at full authority. In a hover field, the collective lever (`fly.lever`, 0 to `lever_max`) holds the ship against the field at `hover_mark`.
- **Speed.** Thrust fades along each axis as the speed that way nears `arm_speed` and never pushes past it; only the boost and jinks go on toward `speed_cap`.
- **Boost and jink.** Boost alone pushes along the nose. Boost with lift, strafe or (Flight Assist off) retro held doubles that thruster with no fade: a jink.
- **Fuel.** The tank pays per m/s of speed change (`fuel_per_dv`), plus a burn for the boost and for going faster than `arm_speed`. The small thrusters pay from the charge (counted in m/s of speed change), which refills while there is fuel; above `small_tank_speed` their push along the velocity pays the tank. Turning is free. At 0 fuel the ship still turns but cannot thrust.
- **Helpers.** `level_out_step` turns the ship level (roll 0, pitch within `level_max_pitch`) while it coasts, snapping level after `level_snap_s`. `impact_point` says where a ship would meet a pad's plane under a constant pull, for a landing ring. `Attitude::quat` returns `[x, y, z, w]`.

- **Turn response (0.2).** By default a turn builds and fades at `rate_accel`, a straight line both ways, so a heavy ship coasts a long way after the stick is let go (`rate_max² / 2·rate_accel`). Set `rate_tau_in` and `rate_tau_out` (seconds, per axis) and the turn instead answers at once, eases up to its rate (still no faster than `rate_accel`) and fades with `rate_tau_out`, coasting only about `rate_max × rate_tau_out`: weighty without overshooting the aim. It applies with Flight Assist on in open space and with it off; the hover assist keeps its own.
- **Bank turn and auto-level (0.2).** `coord_world: true` makes the held-bank turn a turn about world up (the nose sweeps round the horizon and, steeply banked, pulls toward the canopy) instead of a yaw about the deck, which drops the nose at steep banks. `auto_level_hands_off: true` waits for the pitch and yaw sticks to be centred as well as the roll stick before the wings level, so a bank held in a pulled turn stays put.
- **Grip from behind and turning within it (0.2).** By default a course more than 90° off the nose only loses its sideways part, so a ship that pulls harder than its grip can follow, or slides round with `no_grip`, can end up gliding backwards. `grip_behind: true` swings such a course round to the nose too (keeping its speed), so the ship always goes where it points. `turn_within_grip: true` holds pitch and yaw together to `grip / speed`, the rate the grip can swing the course, so a fast ship turns wider instead of sliding off its nose, much as an aircraft's turn widens with speed; with the grip off (`no_grip`, a translation) the nose turns freely.

Every number is in `Engine` (thrusters and charge) or `Tuning` (feel). The defaults are one prototype's first guesses; tune them for your game.

## Limits

- One ship per call, no collisions: the game moves the ship back out of anything it hits and changes `body.vel`.
- `Body.vel` is relative to whatever frame the game flies in. Near a moving pad, pass the velocity relative to the pad and carry the pad's own motion yourself.
- It assumes a fixed tick. It works at any `dt`, but the tests and tuning use 1/60 s.
- `f32` throughout. Far from the origin (tens of kilometres) positions get coarse; keep the game's origin near the player.
- The hover assist takes over the main engine as well as the small thrusters, and pays all its thrust from the charge first.

## Tests

`cargo test` runs 29 tests in a few seconds (no Bevy needed): the attitude stays square over 100,000 random ticks, the thrust table, flip and stop times, speed caps, braking distance, fuel and charge, the hover assist, Flight Assist's grip and auto-level, level-out, and each of 0.2's options against the behaviour it replaces. `tests/flight6.rs` says what each one checks.

## License

CC0 1.0 Universal, see [LICENSE](LICENSE). Carter David Long (Scaffold Games) has dedicated this code to the public domain: use it, change it, sell games with it, no credit needed. A mention is always welcome.

Written with Claude under Carter's direction, first for a ship prototype and then tuned for a space-combat game in development.

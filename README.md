# Synesthaxia.glsl
*A most ambitious ghostty cursor shader / One of the... shaders you've ever seen*.

### The TLDR

Was advised to put this in its own repo instead of spamming the ghostty
Discord's "showcase" channel (when will I learn?).

***Ghosttly-Cursor*** *(WIP shader, final name → **Synesthax**)*

**Why bother?**
• Adapts to **any** colorscheme – samples syntax color at origin & dest (needs `smear-cursor.nvim` with anims off and cursor_color → "none" **or** a color-inheriting terminal cursor).
• Tweened motion mirrors your cursor shape, easing long jumps & markdown-wrap toggles.
• Color cue tells you *what* you landed on.
• Tunables: `DURATION`, edge softness, opacity, etc.

**Pain points**
• `TRAIL_OPACITY` & `EDGE_SOFT` interact weirdly.
• `CURVE_STRENGTH` (–1…+1) no noticeable effect on trail curve.
• Early vars (`CURSOR_HIDE_AT`, `GLOW_INTENSITY`) currently hurt visuals.
• Bugs: top-row color bias; trail sometimes missing, sometimes dest-biased; opacity ignored.

**Roadmap**
▫ *Local text glow* (à la @Utensil and their `cursor_blaze_lightning.glsl`).
▫ *Hide real cursor* during tween + safety timer.
▫ Handle whitespace / undercurl highlights.
▫ *Physics*: inertia overshoot, elastic trail, curved arcs (Disney sing-along vibes).
▫ Auto-scale `DURATION` by jump distance.
▫ Delta-time (using `iFrameRate`
▫ Persistent tween on rapid repeats (use tween pos as temp origin (using `iFrame` and `iTimeCursorChange`?)).

**Crazy Neovim hooks**
— Parse colorscheme → feed exact syntax colors into shader for context-aware FX.
— Mode-check via statusline color → yank/visual/etc. get native GPU animations.

**Alt cursor FX concepts**
• Cursor → lightning bolt → strike destination → reform cursor
• Splits into flock (birds/bats) then reform cursor
• Breaks into line-segments taking unique paths (variations  abound)
• “Orcane” (of *RoA* fame) melt-pop teleport + fading puddle (would pair well w/ `flash.nvim` remote operator mode)


# PRESENTING... A (heavily WIP) cursor shader

## Tentative (eventual) name: `"_Synesthaxia_"`(a portmanteau of `"synesthesia"` and `syntax`):

### Current "shame name": `"_Ghosttly-Cursor_"` (both cursor and tween still visible)

**Pros**:

- Matches ANY colorscheme and picks up the syntax highlighting at cursor origin
  and location. (currently requires `smear-cursor.nvim` or your terminal cursor
  being set up to inherit color from the syntax highlighting at its current
  location.)

- Mimics your cursor shape and tweens the animation between origin and
  destination, further emphasizing the location and direction of your jump.

- Has some utility, informing you on the nature of the location you're moving
  your cursor to. It also helps to deal with the jarring effect of toggling the
  rendering of markdown, for example. The tween animation tracks from the screen
  position before toggle to after, preventing your eyes from getting lost
  looking for the cursor location (really useful when text wrapping is turned on).

- Has several tunable parameters (standard `DURATION`, as well as visual
  adjustments for the tween cursor)

**Cons**:

- Requires `smear-cursor.nvim` (configured with all of the smear animations off.
  Just needed for `require('smear-cursor').setup({ cursor_color = 'none' })`)

- Has several parameters that aren't correctly "realized" in the code:
  - `TRAIL_OPACITY`
    - Interferes with (unwanted) hardcoded trail transparency, or has little to
      no effect.

  - `CURVE_STRENGTH`
    - Doesn't seem to do anything at -1 to +1, and nothing "useful" at
      lower/higher values.
      Should either add a convex scoop or a concave bulge to the trail along
      direction of motion.

  - `EDGE_SOFT`
    - Too sensitive, probably not normalized. Currently set to very low value
      as it currently controls the visual crispness of the tween cursor's edges.

- Several others, like `CURSOR_HIDE_AT` and `GLOW_INTENSITY` are aimed at
  upcoming features, and currently have a negative effect on the visual quality.

- **BUGS!!!**:
  - Sampling biased toward top of cell (always inherits color of undercurls on row
    above)

  - Trail renders inconsistently:
    - (Sometimes not at all)

    - (Sometimes with the intended origin -> destination gradient)

    - (Mostly biased toward destination syntax color (nearly always works))

    - (`TRAIL_OPACITY` not making trail more opaque)

    - Others???

#### **Planned Features**:

- _Local text illumination_ - Following the trend set by == @Utensil == with
  their extraordinarily satisfying `cursor_blaze_lightning.glsl` , I want this
  shader to cast dynamic illumination on the surrounding text -- with our color
  sampling strategy, that illumination will match the syntax highlighting group
  under the cursor.

- _Dynamically hide the actual cursor_ (Requires additional integration) I
  will have the actual cursor vanish for the duration of the tween, then
  reappear as it reaches the destination. This should maintain the illusion that
  the tween is ACTUALLY the cursor. I'll also have to include something to
  recall the tween cursor in the event of something preventing the
  animation from completing within the time specified in the cursor-hiding autocmd.

- _Handling blank characters, whitespace, undercurls, etc_ - Want to be able
  to indicate these in addition to syntax (undercurl on trailing spaces, double
  spaces within a string, etc.)

- _Cursor Physics_ - Allowing for:
  - Simulated inertia (cursor has acceleration curve and overshoots destination
    slightly to imply it has weight (tunable))

  - Springiness (having trail act in an elastic manner (tunable))
    - This can be done already in `smear-cursor.nvim` by messing with damping
      and having the `slowdown_exponent` set to a negative number... but I want
      it in this shader too.

  - Curving trajectories based on movement direction, which will be much more
    visually satisfying/interesting for vertical cursor leaps, especially in
    insert mode. Think 'old-school Disney sing-along lyrics'.
    - (e.g. moving right = arcing traj. over current line)

    - (e.g. moving left = arcing traj. below current line)

    - (These sorts of animations could also include 'anticipation')

- _Consistent tween motion_ - Currently distance travelled has a big impact
  on UX when using this shader. Short jumps feel too slow, while long distance
  jumps feel rushed. I aim to have `DURATION` dynamically adjust to jump
  distance to create a consistent feeling to the animation (standard in other
  cursor shaders I believe, but this one was vibe-coded more or less from scratch)

- _Persistent tweened cursor_ - Once the cursor can be hidden, I will need
  a way, on rapid-fire movement operations (like press-hold on `k`/`Up` or
  `j`/`Down`), to prevent the tween cursor from jumping to each new origin point
  because that jump breaks up the continuity. 'What's your plan?'', you might ask.
  To use the current tween cursor position as a temporary 'origin' point. The
  benefits? I think the syntax highlighting could be dynamically grabbed each
  time a new cursor movement is made, so that the tweened cursor's color matches
  what it hovers on just like the real cursor.

##### _**nvim-specific behavior triggers (WILD AND CRAZY IDEAS)**_

- Okay, so hear me out. Right now I'm reading the syntax colors directly from
  the screen... but what if we set up an autocmd or script to run and read our
  colorscheme file to obtain the actual colors being used for various things.
  I could then write these into the shader file and automatically reload the
  configuration on startup and when the active colorscheme changes.
  If these variables are filled in the shader, they can be used to detect
  exactly what is being highlighted at the target location and respond in X way.
  (e.g. custom cursor behavior for var/num/func/str/comment/header/etc.).

- Mode-detection could be performed by watching the mode indicator in the status
  line, meaning yank/delete/visual motions can have terminal-native shader-based
  animations. Detecting which mode would be as simple as knowing what colors
  were being used for the indicator and checking for that color every frame/other
  frame.

### ==_Other Cursor Shader Ideas_== - Rather than simply tweening a copy of the cursor,

I'm thinking of few different ideas, mostly involving cursor decomposition,
but also some more exotic selections, like:

- (Cursor Decomposition) - Cursor becomes lightning and arcs from origin to
  destination and reconstituting the cursor (inspired by Utensil's
  `cursor_blaze_lightning.glsl`)

- (Cursor Decomposition) - Cursor breaks apart into birds/bats and (flocking
  algorithm) navigate their way to the cursor destination (the decomposition
  and reconstitution would happen fast, with 'straggler' members of the flock
  rejoining over time (or getting recycled if a maximum number is hit).

- (Cursor Decomposition) - Cursor splits into individual line segments that
  take separate paths to the target (this one has a lot of potential I think).

- (Emphasised Teleportation) - Instead of tweening at all, why not make things
  more interesting? Look up "Orcane" (Rivals of Aether) and the way their
  puddle works, both mechanically and visually. I think a shader where your
  cursor melts into the 'floor' of the line you're on and emerges from beneath
  the target position sounds cool as hell. Maybe it could leave a temporary
  'puddle' at the origin that fades the longer could work will with
  `flash.nvim`s remote operator mode.

- (Cursor 'Wake') - cursor sends 'ripple'/'wave' distortions outward
  perpindicular to the direction of motion. Wake propagates and has a limited
  number of bounces off terminal edges.

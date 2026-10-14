<body>
  <div id="notifOverlay" style="position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,31,63,0);transition:background 0.5s ease;z-index:9998;display:none;">
    <div id="notif" style="background:rgba(255,255,255,0.9);-webkit-backdrop-filter:blur(8px);backdrop-filter:blur(8px);border:1px solid rgba(0,31,63,0.25);color:#111;padding:26px 44px 22px;border-radius:8px;border-top:4px solid #001f3f;text-align:center;box-shadow:0 6px 24px rgba(0,0,0,0.25);width:max-content;max-width:92vw;box-sizing:border-box;position:fixed;top:50%;left:50%;transform:translate(-50%,-44%);opacity:0;transition:opacity 0.5s ease,transform 0.5s ease;margin:0;text-wrap:balance;">
      <span onclick="closeNotif()" style="position:absolute;top:8px;right:14px;font-size:16px;cursor:pointer;color:#666;" aria-label="Close">✕</span>
      <div style="font-size:20px;font-weight:600;color:#001f3f;margin-bottom:8px;">On the academic job market, 2026–27</div>
      <div style="font-size:16px;line-height:1.5;">Seeking tenure-track faculty and postdoctoral positions</div>
      <div style="margin-top:16px;">
        <a href="research.html" style="display:inline-block;margin:0 4px;padding:6px 14px;border-radius:5px;background:#001f3f;color:#fff;font-size:14px;font-weight:600;text-decoration:none;">Research</a>
        <a href="assets/Ghazal_Abdollahi_CV.pdf" style="display:inline-block;margin:0 4px;padding:5px 14px;border-radius:5px;border:1px solid #001f3f;color:#001f3f;font-size:14px;font-weight:600;text-decoration:none;">Short CV</a>
      </div>
    </div>
  </div>
  <script>
    (function () {
      var overlay = document.getElementById('notifOverlay');
      var card = document.getElementById('notif');
      window.closeNotif = function () {
        window.popupClosed = true;
        card.style.opacity = '0';
        card.style.transform = 'translate(-50%,-44%)';
        overlay.style.background = 'rgba(0,31,63,0)';
        setTimeout(function () { overlay.style.display = 'none'; }, 500);
      };
      overlay.addEventListener('click', function (e) { if (e.target === overlay) closeNotif(); });
      document.addEventListener('keydown', function (e) { if (e.key === 'Escape') closeNotif(); });
      window.addEventListener('load', function () {
        setTimeout(function () {
          overlay.style.display = 'block';
          setTimeout(function () {
            card.style.opacity = '1';
            card.style.transform = 'translate(-50%,-50%)';
            overlay.style.background = 'rgba(0,31,63,0.08)';
          }, 30);
        }, 500);
      });
    })();
  </script>
  <canvas id="bgPattern" aria-hidden="true" style="position:fixed;top:0;left:0;width:100%;height:100%;z-index:-1;pointer-events:none;"></canvas>
  <script>
    (function () {
      // Background: sparse faint dots in the left and right page margins only,
      // never behind the content. It acts out the "Between A and B" story.
      // Each margin runs its own slow cycle:
      //   1. two dots light up (point A and point B), near or far apart;
      //   2. a connection is tried from one toward the other, one hop at a time.
      //      Before each hop, the dots within reach glow softly: the little that
      //      can be seen from where the line stands. Then it picks one;
      //   3. often a try reaches a dead end. It pauses there (a small ring
      //      pulses, unsure), then fades, leaving a faint trace of what it
      //      learned, and the next try avoids that dead end. This can happen up
      //      to three times; sometimes the first try already works;
      //   4. the try that reaches the other dot glows gently and settles into a
      //      very faint trace. Traces of earlier paths linger for a while, so
      //      over time many paths add up to a pattern. Then a new pair is chosen.
      // It starts once the status popup has been closed. The margins follow the
      // window size; on a narrow screen with no margins nothing is drawn.
      var canvas = document.getElementById('bgPattern');
      if (!canvas || !canvas.getContext) return;
      var ctx = canvas.getContext('2d');
      var GAP = 110, REACH = 210, MAX_FAILS = 3, QUICK = 0.3;
      var ENDS = 900, LOOK = 380, STEP = 480, STALL = 900, GIVE_UP = 650, SHINE = 1300, FADE = 1400;
      var HOP = LOOK + STEP, MEMORY = 120000, MEMORY_MAX = 8;
      var still = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
      var w = 0, h = 0, pts = [], eps = [], memory = [];
      var lastFrame = 0, began = false, nextAt = { '-1': 0, '1': 0 };
      function fill(from, to, sideId) {
        // Spread dots evenly over one margin, however wide or narrow it is.
        var width = to - from;
        if (width < 36) return;
        var cols = Math.max(1, Math.round(width / GAP));
        var colW = width / cols;
        var sway = Math.max(0, Math.min(GAP * 0.35, colW / 2 - 10));
        for (var c = 0; c < cols; c++) {
          for (var y = GAP / 2; y < h; y += GAP) {
            pts.push({
              x: from + (c + 0.5) * colW + (Math.random() * 2 - 1) * sway,
              y: y + (Math.random() - 0.5) * GAP * 0.7,
              side: sideId
            });
          }
        }
      }
      function build() {
        // Dots live only in the space left and right of the content column. As
        // the window narrows the margins shrink, and once they are gone (tablet
        // and phone widths) there are no dots at all.
        pts = [];
        eps = [];
        memory = [];
        var column = document.querySelector('.wrapper');
        if (!column) return;
        var r = column.getBoundingClientRect();
        fill(0, r.left - 30, -1);
        fill(r.right + 30, w, 1);
      }
      function pick(list) { return list[Math.floor(Math.random() * list.length)]; }
      function gapBetween(i, j) {
        return Math.sqrt(Math.pow(pts[i].x - pts[j].x, 2) + Math.pow(pts[i].y - pts[j].y, 2));
      }
      function makeRoute(from, to, side, avoid) {
        // Wander from one end dot to the other in any direction, each hop
        // landing a little closer to the target. Dead ends found by earlier
        // tries are avoided. For each hop, the dots that were within reach
        // (the options) are kept so they can glow before the choice is made.
        var cur = from, route = [from], opts = [], used = {};
        used[from] = true;
        for (var guard = 0; guard < 40; guard++) {
          var left = gapBetween(cur, to), near = [];
          for (var j = 0; j < pts.length; j++) {
            if (j === to || used[j] || avoid[j] || pts[j].side !== side) continue;
            if (gapBetween(cur, j) <= REACH && gapBetween(j, to) < left - GAP * 0.15) near.push(j);
          }
          var canFinish = left <= REACH * 1.15;
          if (canFinish && (!near.length || (route.length > 1 && Math.random() < 0.5))) {
            opts.push(near.concat([to]));
            route.push(to);
            return { path: route, opts: opts };
          }
          if (!near.length) return null;
          opts.push(canFinish ? near.concat([to]) : near);
          cur = pick(near);
          used[cur] = true;
          route.push(cur);
        }
        return null;
      }
      function startEpisode(t, side) {
        var j, mine = [], far = [];
        for (j = 0; j < pts.length; j++) if (pts[j].side === side) mine.push(j);
        if (mine.length < 3) return false;
        // A random first dot anywhere in this margin, and a second one a short,
        // medium or long way off in any direction.
        var from = pick(mine), roll = Math.random();
        var lo = roll < 0.35 ? 1.4 : (roll < 0.75 ? 2.6 : 4.5);
        var hi = roll < 0.35 ? 2.6 : (roll < 0.75 ? 4.5 : 99);
        for (j = 0; j < mine.length; j++) {
          var d = gapBetween(from, mine[j]) / GAP;
          if (d >= lo && d < hi) far.push(mine[j]);
        }
        if (!far.length) return false;
        var to = pick(far);
        // Sometimes the decision is quick (no failed tries); otherwise one to
        // three different ways are tried and dropped first.
        var fails = Math.random() < QUICK ? 0 : 1 + Math.floor(Math.random() * MAX_FAILS);
        var tries = [], seen = {}, avoid = {}, clock = ENDS;
        for (var n = 0; n < 30 && tries.length < fails + 1; n++) {
          var r = makeRoute(from, to, side, avoid);
          if (!r || r.path.length < 3 || seen[r.path.join(',')]) continue;
          seen[r.path.join(',')] = true;
          var full = r.path.length - 1;
          var failed = tries.length < fails;
          // A failed try stops at a dead end part of the way along, and the
          // next tries learn to stay away from that dot.
          var hops = failed ? 1 + Math.floor(Math.random() * (full - 1)) : full;
          if (failed) avoid[r.path[hops]] = true;
          var reached = clock + hops * HOP;
          tries.push({ route: r.path, opts: r.opts, hops: hops, full: full, start: clock, reached: reached });
          clock = reached + STALL + GIVE_UP + 200;
        }
        if (!tries.length) return false;
        // If no try got all the way, let the last one finish.
        var win = tries[tries.length - 1];
        if (win.hops < win.full) { win.hops = win.full; win.reached = win.start + win.full * HOP; }
        eps.push({ t0: t, side: side, from: from, to: to, tries: tries });
        return true;
      }
      function clamp(v) { return v < 0 ? 0 : (v > 1 ? 1 : v); }
      function dot(pt, radius, alpha) {
        ctx.fillStyle = 'rgba(0,31,63,' + alpha + ')';
        ctx.beginPath();
        ctx.arc(pt.x, pt.y, radius, 0, 6.2832);
        ctx.fill();
      }
      function ease(v) { return 0.5 - 0.5 * Math.cos(3.1416 * clamp(v)); }
      function drawTry(tr, age, alpha, extra, looking) {
        for (var k = 0; k < tr.hops; k++) {
          var hopStart = tr.start + k * HOP;
          if (age < hopStart) break;
          if (looking) {
            // Before the hop, the dots within reach glow while one is chosen.
            var lp = (age - hopStart) / (LOOK + STEP * 0.6);
            if (lp < 1) {
              var g = Math.sin(3.1416 * lp), o = tr.opts[k];
              for (var q = 0; q < o.length; q++) dot(pts[o[q]], 1.3 + 1.1 * g, 0.08 + 0.2 * g);
            }
          }
          var grow = ease((age - hopStart - LOOK) / STEP);
          if (grow <= 0) break;
          var A = pts[tr.route[k]], B = pts[tr.route[k + 1]];
          ctx.lineWidth = 1 + 0.5 * extra;
          ctx.strokeStyle = 'rgba(0,31,63,' + alpha + ')';
          ctx.beginPath();
          ctx.moveTo(A.x, A.y);
          ctx.lineTo(A.x + (B.x - A.x) * grow, A.y + (B.y - A.y) * grow);
          ctx.stroke();
          if (grow > 0.99) dot(B, 2 + 0.6 * extra, alpha * 1.2);
        }
      }
      function trace(route, alpha) {
        // A settled path from an earlier episode, drawn very faintly.
        ctx.lineWidth = 1;
        ctx.strokeStyle = 'rgba(0,31,63,' + alpha + ')';
        for (var k = 0; k < route.length - 1; k++) {
          var A = pts[route[k]], B = pts[route[k + 1]];
          ctx.beginPath();
          ctx.moveTo(A.x, A.y);
          ctx.lineTo(B.x, B.y);
          ctx.stroke();
          dot(B, 2, alpha * 1.2);
        }
      }
      function draw(t) {
        ctx.clearRect(0, 0, w, h);
        for (var i = 0; i < pts.length; i++) dot(pts[i], 1.3, 0.08);
        ctx.lineCap = 'round';
        // Many earlier paths, side by side: slowly they add up to a pattern.
        for (var m = memory.length - 1; m >= 0; m--) {
          var life = 1 - clamp((t - memory[m].born) / MEMORY);
          if (life <= 0) { memory.splice(m, 1); continue; }
          trace(memory[m].route, 0.06 * life);
        }
        for (var e = eps.length - 1; e >= 0; e--) {
          var ep = eps[e], age = t - ep.t0;
          var win = ep.tries[ep.tries.length - 1];
          var shineStart = win.reached + 250;
          if (age > shineStart + SHINE + FADE) {
            // The kept path settles into a faint trace that lingers.
            memory.push({ route: win.route, side: ep.side, born: t });
            var mine = 0;
            for (var c = memory.length - 1; c >= 0; c--) {
              if (memory[c].side === ep.side && ++mine > MEMORY_MAX) memory.splice(c, 1);
            }
            nextAt[ep.side] = t + 900 + Math.random() * 1300;
            eps.splice(e, 1);
            continue;
          }
          var shine = age > shineStart ? Math.sin(3.1416 * clamp((age - shineStart) / SHINE)) : 0;
          var keep = 1 - clamp((age - shineStart - SHINE) / FADE);
          for (var r = 0; r < ep.tries.length - 1; r++) {
            var tr = ep.tries[r];
            if (age < tr.start) continue;
            // A dead end: pause there, unsure, then let it go. A faint trace
            // stays for the rest of the episode, a little more of the map.
            var drop = 1 - clamp((age - tr.reached - STALL) / GIVE_UP);
            drawTry(tr, age, (0.035 + 0.12 * drop) * keep, 0, age < tr.reached);
            if (age >= tr.reached && drop > 0) {
              var p = (age - tr.reached) / STALL, end = pts[tr.route[tr.hops]];
              ctx.lineWidth = 1;
              ctx.strokeStyle = 'rgba(0,31,63,' + (0.25 * drop * clamp(p * 3)) + ')';
              ctx.beginPath();
              ctx.arc(end.x, end.y, 5 + 1.5 * Math.sin(6.2832 * p), 0, 6.2832);
              ctx.stroke();
            }
          }
          if (age >= win.start) drawTry(win, age, 0.06 + (0.1 + 0.12 * shine) * keep, shine, age < win.reached);
          var appear = clamp(age / 500) * keep;
          dot(pts[ep.from], 3 + 0.6 * shine, (0.3 + 0.1 * shine) * appear);
          dot(pts[ep.to], 3 + 0.6 * shine, (0.3 + 0.1 * shine) * appear);
        }
      }
      function busy(side) {
        for (var e = 0; e < eps.length; e++) if (eps[e].side === side) return true;
        return false;
      }
      function resize() {
        var ratio = window.devicePixelRatio || 1;
        w = window.innerWidth;
        h = window.innerHeight;
        canvas.width = w * ratio;
        canvas.height = h * ratio;
        ctx.setTransform(ratio, 0, 0, ratio, 0, 0);
        build();
        draw(0);
      }
      function tick(t) {
        if (t - lastFrame > 30) {
          lastFrame = t;
          if (window.popupClosed) {
            // The right margin starts a little after the left one, so the two
            // are never in step.
            if (!began) { began = true; nextAt[-1] = t; nextAt[1] = t + 2600; }
            if (!busy(-1) && t >= nextAt[-1]) startEpisode(t, -1);
            if (!busy(1) && t >= nextAt[1]) startEpisode(t, 1);
          }
          draw(t);
        }
        window.requestAnimationFrame(tick);
      }
      window.addEventListener('resize', resize);
      resize();
      if (!still) window.requestAnimationFrame(tick);
    })();
  </script>
  <style>
    /* Data story: a small corner button that opens "Between A and B", told one hop at a time. */
    #storyBtn{position:fixed;right:20px;bottom:20px;width:46px;height:46px;border-radius:50%;padding:0;background:rgba(255,255,255,0.92);border:1px solid rgba(0,31,63,0.28);box-shadow:0 3px 12px rgba(0,31,63,0.14);cursor:pointer;z-index:9000;display:flex;align-items:center;justify-content:center;transition:transform .25s ease,box-shadow .25s ease;}
    #storyBtn:hover,#storyBtn:focus-visible{transform:translateY(-2px);box-shadow:0 6px 18px rgba(0,31,63,0.22);outline:none;}
    #storyBtn .tip{position:absolute;right:56px;white-space:nowrap;background:#001f3f;color:#fff;font-size:13px;padding:5px 10px;border-radius:5px;opacity:0;pointer-events:none;transition:opacity .2s ease;}
    #storyBtn:hover .tip,#storyBtn:focus-visible .tip{opacity:1;}
    #storyOverlay{position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,31,63,0);transition:background .4s ease;z-index:9999;display:none;}
    #storyOverlay.open{background:rgba(0,31,63,0.12);}
    #storyCard{position:fixed;top:50%;left:50%;width:460px;max-width:calc(100vw - 32px);box-sizing:border-box;transform:translate(-50%,-46%);opacity:0;transition:opacity .4s ease,transform .4s ease;background:rgba(255,255,255,0.95);-webkit-backdrop-filter:blur(8px);backdrop-filter:blur(8px);border:1px solid rgba(0,31,63,0.2);border-top:4px solid #001f3f;border-radius:8px;box-shadow:0 8px 30px rgba(0,31,63,0.2);padding:26px 28px 20px;color:#222;text-align:left;}
    #storyOverlay.open #storyCard{opacity:1;transform:translate(-50%,-50%);}
    #storyCard .x{position:absolute;top:8px;right:14px;font-size:16px;cursor:pointer;color:#666;background:none;border:0;padding:4px;}
    #storyCard .kicker{font-size:12px;letter-spacing:.12em;text-transform:uppercase;color:#6b7a8c;margin-bottom:10px;}
    #storyArt{display:block;width:100%;height:auto;margin:0 0 14px;}
    #storyArt .g-dots{opacity:0;transition:opacity .7s ease;}
    #storyArt .g-try,#storyArt .g-path{fill:none;stroke:#001f3f;stroke-linecap:round;stroke-linejoin:round;stroke-dasharray:1 1;stroke-dashoffset:1;transition:stroke-dashoffset 1.4s ease,opacity .8s ease,stroke-width .8s ease;}
    #storyArt .g-try{stroke-width:1.5;opacity:.55;}
    #storyArt .g-path{stroke-width:1.8;}
    #storyArt .g-end{opacity:0;transition:opacity .6s ease .9s;}
    #storyArt.a1 .g-dots{opacity:1;}
    #storyArt.a2 .g-try{stroke-dashoffset:0;}
    #storyArt.a2 .g-end{opacity:1;}
    #storyArt.a3 .g-try,#storyArt.a3 .g-end{opacity:.14;transition-delay:0s;}
    #storyArt.a4 .g-path{stroke-dashoffset:0;}
    #storyArt.a5 .g-path{stroke-width:2.8;}
    #storyArt .g-many{fill:none;stroke:#001f3f;stroke-width:1.2;stroke-linecap:round;stroke-linejoin:round;opacity:0;transition:opacity 1.2s ease;}
    #storyArt.a6 .g-many{opacity:.22;}
    #storyArt.a6 .g-path{stroke-width:2;}
    #storyText{transition:opacity .2s ease;min-height:132px;}
    #storyText .t{font-size:19px;font-weight:600;color:#001f3f;margin:0 0 6px;}
    #storyText .p{font-size:15.5px;line-height:1.6;margin:0;}
    #storyText .note{font-size:13.5px;line-height:1.5;color:#6b7a8c;font-style:italic;margin:10px 0 0;}
    #storyNav{display:flex;align-items:center;justify-content:space-between;margin-top:16px;}
    #storyHops{display:flex;align-items:center;}
    #storyHops span{width:7px;height:7px;border-radius:50%;background:rgba(0,31,63,0.18);transition:background .3s ease;}
    #storyHops span.on{background:#001f3f;}
    #storyHops i{width:12px;height:1px;background:rgba(0,31,63,0.18);transition:background .3s ease;}
    #storyHops i.on{background:#001f3f;}
    #storyNav button{font:inherit;font-size:14px;font-weight:600;border-radius:5px;cursor:pointer;padding:6px 14px;}
    #storyBack{background:none;border:0;color:#001f3f;}
    #storyNext{background:#001f3f;color:#fff;border:1px solid #001f3f;margin-left:6px;}
    @media (prefers-reduced-motion: reduce){#storyBtn,#storyCard,#storyArt *,#storyText,#storyOverlay{transition:none !important;}}
  </style>
  <button id="storyBtn" type="button" aria-label="Read a short story: Between A and B">
    <svg width="24" height="24" viewBox="0 0 24 24" aria-hidden="true"><line x1="4" y1="18" x2="13" y2="21" stroke="#001f3f" stroke-width="1.3" stroke-dasharray="2 2" opacity=".45"/><polyline points="4,18 11,7 20,12" fill="none" stroke="#001f3f" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/><circle cx="4" cy="18" r="2.4" fill="#001f3f"/><circle cx="11" cy="7" r="1.7" fill="#001f3f"/><circle cx="20" cy="12" r="2.4" fill="#001f3f"/></svg>
    <span class="tip">Between A and B: a short story</span>
  </button>
  <div id="storyOverlay">
    <div id="storyCard" role="dialog" aria-modal="true" aria-labelledby="storyTitle">
      <button class="x" type="button" id="storyClose" aria-label="Close">✕</button>
      <div class="kicker">A data story by Ghazal · Between A and B</div>
      <svg id="storyArt" viewBox="0 0 300 96" aria-hidden="true">
        <g class="g-dots" fill="rgba(0,31,63,0.35)"><circle cx="80" cy="30" r="2.6"/><circle cx="100" cy="74" r="2.6"/><circle cx="150" cy="58" r="2.6"/><circle cx="165" cy="22" r="2.6"/><circle cx="212" cy="72" r="2.6"/><circle cx="232" cy="38" r="2.6"/><circle cx="125" cy="40" r="2.6"/></g>
        <g class="g-many"><polyline points="22,60 100,74 212,72 278,44"/><polyline points="22,60 125,40 232,38 278,44"/><polyline points="22,60 80,30 150,58 232,38 278,44"/><polyline points="22,60 100,74 125,40 165,22 278,44"/><polyline points="22,60 150,58 212,72 278,44"/></g>
        <polyline class="g-try" pathLength="1" points="22,60 100,74 150,58"/>
        <circle class="g-end" cx="150" cy="58" r="4.5" fill="none" stroke="#001f3f" stroke-width="1.2"/>
        <polyline class="g-path" pathLength="1" points="22,60 80,30 165,22 232,38 278,44"/>
        <circle cx="22" cy="60" r="5" fill="#001f3f"/><circle cx="278" cy="44" r="5" fill="#001f3f"/>
        <text x="22" y="84" font-size="11" fill="#001f3f" text-anchor="middle" font-weight="600">A</text>
        <text x="278" y="30" font-size="11" fill="#001f3f" text-anchor="middle" font-weight="600">B</text>
      </svg>
      <div id="storyText" aria-live="polite"><div class="t" id="storyTitle"></div><p class="p"></p><p class="note"></p></div>
      <div id="storyNav">
        <div id="storyHops" aria-hidden="true"></div>
        <div><button id="storyBack" type="button">← Back</button><button id="storyNext" type="button">Next →</button></div>
      </div>
    </div>
  </div>
  <script>
    (function () {
      // "Between A and B": the same idea the margin dots act out, told one hop at a time.
      // The first step is the intro in my own voice; the drawing shows the finished path,
      // then clears and replays from step two on.
      var steps = [
        { t: 'Why the dots?', p: 'Hi, I’m Ghazal. You may have noticed the faint dots in the margins of this page. Two of them light up, and a line tries to reach from one to the other. Sometimes it stops, fades, and tries another way, until one path finally connects. I put them there because that is how my research and my life have felt to me: rarely a straight line, and always one hop at a time. Here is the story behind them.' },
        { t: 'Point A, point B.', p: 'We usually know where we stand, and roughly where we want to be. The line between the two is almost never straight.' },
        { t: 'Dots in between.', p: 'Between here and there are milestones, people, and small choices. Nobody picks the whole route at once. We only ever pick the next hop, using the little we can see from where we stand.' },
        { t: 'A hop that doesn’t hold.', p: 'Sometimes we head toward a dot, and halfway there it stops making sense. A dead end. We pause, unsure where to go next.' },
        { t: 'Thinking it again.', p: 'So we step back and look again. In machine learning, a wrong guess is not a failure. It is the error a model learns from. The path we let go of did the same for us: it showed us a little more of the map.' },
        { t: 'The next hop.', p: 'Then we choose. Not because we are certain, but because we have looked enough. One dot, then the next.' },
        { t: 'Looking back.', p: 'When the line finally connects, it looks as if the path was always there. It wasn’t. It was made one doubt and one decision at a time.' },
        { t: 'Seen from above.', p: 'One path tells you very little. Lay many of them side by side, with their hesitations, detours, and dead ends, and a shape appears. That is how machine learning reads behavior: not one choice at a time, but the pattern that many choices form. Our lives draw that kind of pattern too. We just see it one hop at a time.', note: 'You may have felt this in your own life too: a moment when the detours and dead ends suddenly made sense, and the path you were on finally lit up. ~ Ghazal' }
      ];
      var btn = document.getElementById('storyBtn'), overlay = document.getElementById('storyOverlay');
      var art = document.getElementById('storyArt'), text = document.getElementById('storyText');
      var title = text.querySelector('.t'), para = text.querySelector('.p'), note = text.querySelector('.note');
      var hops = document.getElementById('storyHops'), back = document.getElementById('storyBack'), next = document.getElementById('storyNext');
      var i = 0, isOpen = false, swap = null;
      for (var k = 0; k < steps.length; k++) hops.innerHTML += (k ? '<i></i>' : '') + '<span></span>';
      function render(fade) {
        var cls = i === 0 ? ['a1', 'a4'] : [];
        for (var k = 1; k < i; k++) cls.push('a' + k);
        art.setAttribute('class', cls.join(' '));
        var marks = hops.children;
        for (var m = 0; m < marks.length; m++) marks[m].className = (m <= i * 2) ? 'on' : '';
        back.style.visibility = i ? 'visible' : 'hidden';
        next.textContent = i === steps.length - 1 ? 'Close' : 'Next →';
        clearTimeout(swap);
        text.style.opacity = fade ? '0' : '1';
        swap = setTimeout(function () {
          title.textContent = steps[i].t;
          para.textContent = steps[i].p;
          note.textContent = steps[i].note || '';
          note.style.display = steps[i].note ? 'block' : 'none';
          text.style.opacity = '1';
        }, fade ? 200 : 0);
      }
      function openStory() {
        i = 0; isOpen = true;
        overlay.style.display = 'block';
        render(false);
        setTimeout(function () { overlay.classList.add('open'); next.focus(); }, 30);
      }
      function closeStory() {
        isOpen = false;
        overlay.classList.remove('open');
        setTimeout(function () { if (!isOpen) overlay.style.display = 'none'; }, 400);
        btn.focus();
      }
      function go(d) {
        if (i + d >= steps.length) return closeStory();
        if (i + d < 0) return;
        i += d;
        render(true);
      }
      btn.addEventListener('click', openStory);
      document.getElementById('storyClose').addEventListener('click', closeStory);
      next.addEventListener('click', function () { go(1); });
      back.addEventListener('click', function () { go(-1); });
      overlay.addEventListener('click', function (e) { if (e.target === overlay) closeStory(); });
      document.addEventListener('keydown', function (e) {
        if (!isOpen) return;
        if (e.key === 'Escape') closeStory();
        if (e.key === 'ArrowRight') go(1);
        if (e.key === 'ArrowLeft') go(-1);
      });
    })();
  </script>
</body>
# PhD Candidate in Computer Science
[University of Utah](https://www.utah.edu/) | [Flux Research Group](https://www.flux.utah.edu/)

[![Google Scholar](https://img.shields.io/badge/Google%20Scholar-4285F4?style=flat&logo=google-scholar&logoColor=white)](https://scholar.google.com/citations?hl=en&user=d8byHrIAAAAJ)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/ghazal-abdollahi-b3206a2a9/)



## About

I'm a PhD candidate at the [Kahlert School of Computing](https://www.cs.utah.edu/) at the University of Utah (since 2022), where I work with [Prof. Robert Ricci](https://ricci.io) as part of the [Flux Research Group](https://www.flux.utah.edu/). My research is about how computing systems can notice what is forming in their own data early enough to act on it, in a way people can follow. I call this Interpretable Signal-to-Control (O→S→C: observation, signal, control), and I have developed it in serverless computing, data center networking, and network security.

<div style="border-left:4px solid #001f3f;background:#f4f6f9;padding:12px 16px;margin:20px 0 16px;border-radius:0 6px 6px 0;text-align:left;line-height:1.5;">
  <strong style="color:#001f3f;">On the academic job market, 2026–27</strong><br>
  Seeking tenure-track faculty and postdoctoral positions.
</div>
<div style="display:flex;flex-wrap:wrap;gap:10px;margin:0 0 24px;">
  <a href="research.html" style="display:inline-block;padding:8px 16px;border-radius:5px;background:#001f3f;color:#fff;font-weight:600;text-decoration:none;">Research Overview →</a>
  <a href="assets/Ghazal_Abdollahi_CV.pdf" style="display:inline-block;padding:7px 16px;border-radius:5px;border:1px solid #001f3f;color:#001f3f;font-weight:600;text-decoration:none;">Short CV (PDF)</a>
</div>

<div align="center">
  <img src="assets/MEB.JPG" width="400">
  <br>
  <em>Photo taken at MEB, my office building, Summer 2026.</em>
</div>

## ✉️ Contact

**Email**: Ghazal [dot] Abdollahi [at] utah.edu

I'm open to research collaborations in my research areas. Feel free to reach out through my email!

**Status:** seeking tenure-track faculty and postdoctoral positions.

-------------------------------------------------
## 💡 What My Papers Have Taught Me

- **Serverless:** cold starts and idle containers are one problem, not two. Reading how long a function runs and how soon it will be called again lets a platform steady both at once, with rules an operator can read line by line.
- **Data center networking:** flow populations show structure before any single flow can be classified. Following that idea to its limit led me to the "reactive wall", a negative result that taught me more than the cases where the method worked.
- **Security:** SSH logs hold millions of attempts and little direct evidence of intent. Looking at attacker populations instead of single events reveals who is attacking, and when independently trained learners disagree about an attacker, that disagreement is itself an early sign of a campaign in transition.
- **Across all three:** the useful signals took similar shapes in fields that rarely cite one another, which is the question I most want to keep exploring.

The bigger picture is in the [Research Overview](research.md).

-------------------------------------------------
## 📄 Publications

- **Vertex: Towards Recommender Firewalls Resilient to Post-Detection Evasion** — poster, ACM IMC 2026
- **EPIC: When Learners Disagree, Campaigns Speak; Perceiving Structure in Unfolding SSH Episodes** — poster, ACM SIGCOMM 2026
- **Combo: Proactive Momentum and the Reactive Wall** — ACM NetNeg (Workshop on Negative Results in Networking), SIGCOMM 2026
- **WarmFlex: Coordinated Stabilization of Cold-Start Frequency and Idle Containers in Serverless Platforms** — IEEE SmartCloud 2026
- **NOD: Uncovering Intense Attackers' Behavior Through Nested Outlier Detection from SSH Logs** — PRISM Workshop, NDSS 2026
- **Atomic Resolution Mapping and Electrochemical Analysis of Illicit Drug-Aptamer Complexes and Their Terahertz Signatures** — IEEE NMDC 2025
- **FMap: A Fuzzy Map for Scheduling Elephant Flows Through Jumping** — Concurrency and Computation: Practice and Experience
- **Flow-Aware Forwarding in SDN Data Centers Using a Knapsack-PSO-Based Solution** — IEEE Transactions on Network and Service Management

Published papers are also on [Google Scholar](https://scholar.google.com/citations?hl=en&user=d8byHrIAAAAJ).

-------------------------------------------------
## 🎓 Education

- **Ph.D. candidate in Computer Science**, University of Utah, Salt Lake City, UT (Fall 2022 – present). Kahlert School of Computing, Flux Research Group. Advisor: [Prof. Robert Ricci](https://ricci.io).
- **M.Sc. in Information Technology Engineering – Computer Networks**, Salman Institute of Higher Education, Mashhad, Iran
- **B.Sc. in Information Technology Engineering**, Payame Noor University, Iran

## 🏅 Grants & Awards

- **N2Women Travel Grant**, ACM MobiCom 2026, Austin, TX
- **Conference Travel Grants**: ACM RecSys 2026, USENIX PEPR '26, NSDI 2025, KubeCon 2024, and ACM CCS 2024

## 🔬 Experience

- **ML-based Biomedical Detection Intern**, ECE Department, University of Utah (Summer 2024, Summer 2026). Supervisor: [Prof. Massood Tabib-Azar](https://profiles.faculty.utah.edu/u0640514), [AMANDA Lab](https://amanda.ece.utah.edu/people/). I built machine-learning classification for a wearable THC biosensor and brought the model onto a small, resource-limited microcontroller so that it runs on the device itself.

## 🧑‍🏫 Teaching

**Teaching Assistant**, CS 4000 – Senior Capstone Design, University of Utah (Fall 2026). Instructors: Prof. H. James de St. Germain, Prof. David Bean, and Prof. Fengjiao Wang.

Before the semester began I read and ranked close to 200 project proposals for a class of 224 seniors. Then ten of those proposals became my teams, and the ideas had faces. I meet each team for thirty minutes every week and bring the questions I ask of my own research: Does the data this project depends on actually exist? What is new here that existing applications do not already offer? What will be real by the next milestone? Some of the projects have outside sponsors, including WebBank and the Huntsman Cancer Institute, which connects the work to a purpose beyond the classroom.

What these weeks have taught me about teaching:

- **Ask before telling.** A good question leaves the thinking with the team. Some students open up with more questions, and others need fewer of them and more room.
- **Say what counts before the work starts.** With 224 students and five graders, clear expectations are what make feedback fair.
- **Judge a project by what it is becoming.** Teams spend most of a semester with nothing finished to show, and an early verdict can end an idea that was about to become good.
- **Teach the layer above the prototype.** A team can reach a prototype quickly today, so my time goes to what a tool cannot generate: data, novelty, and scope.

**Courses I would love to teach:** Data Mining, Machine Learning, Advanced Artificial Intelligence, Network Security, Computer Networks, and Cloud and Distributed Systems. I would also like to build a project-based course in which students collect their own data and defend what it can and cannot support.

## 🤝 Service

- **Volunteer Reviewer**: IEEE Transactions on Network and Service Management
- **Volunteer Reviewer**: Future Generation Computer Systems (Elsevier)
- **Artifact Evaluation**: ACM SIGMETRICS 2026 (shadow reviewer)
- **Artifact Evaluation**: IEEE ICPE
- **Student Volunteer**: PMAPS 2026, Salt Lake City, UT

-------------------------------------------------
## 🌐 Conferences & Workshops

Conferences, workshops, and summits I have **presented/attended** at:
- **ACM MobiCom 2026** — Austin, TX, USA *(N2Women Travel Grant)*
- **ACM RecSys 2026** — Minneapolis, Minnesota, USA
- **PMAPS 2026** — Salt Lake City, UT, USA *(Student Volunteer)*
- **SIGCOMM 2026** — Denver, Colorado, USA
- **AI Convergence Summit 2026** — University of Utah, SLC, UT, USA 
- **USENIX PEPR '26** — Santa Clara, CA, USA
- **IEEE SmartCloud 2026** — NYC, New York City, USA
- **NDSS 2026** — San Diego, CA, USA
- **ACM HotNets 2025** — University of Maryland, MD, USA  
- **NSDI 2025** — Philadelphia, PA, USA  
- **AI Summit 2025** — University of Utah, Salt Lake City, UT, USA  
- **KubeCon 2024** — Salt Lake City, UT, USA  
- **ACM CCS 2024** — Salt Lake City, UT, USA  
- **ML & Big Data HPC Workshop** — INSCC, University of Utah, USA

-------------------------------------------------
## 🛠 Key Skills

Cloud and serverless computing · container orchestration · cold-start and idle-resource optimization · data center networking · software-defined networking (SDN) · flow scheduling and forwarding · AI/ML for networking and security · interpretable/explainable AI · anomaly and intrusion detection · attacker behavior and security log analysis · workload characterization · embedded and edge machine learning

-------------------------------------------------
## 🎣 A Quote I Keep Close

I love how this quote makes sense in my research process:
  "It goes always, bit by bit. You don't see the whole thing at once.
  The way it unfolds is anybody's guess. It's kind of a magical thing.
  But eventually, it gets there. If you focus on it, it'll draw other
  ideas in. And... but you don't know. Like fishing, you have to have
  patience. You don't know how long it'll take. But if you keep
  focusing on it, it'll... and wiggle the line a little bit, it'll
  bring them in. Intuition is the intellect and emotions swimming
  together." — David Lynch

-------------------------------------------------
## 📍 Salt Lake City, Utah, USA

Salt Lake City is a quiet mountain city surrounded by peaceful scenery and genuinely nice people. You’re only minutes to hours from spectacular landscapes; national parks like [Yellowstone](https://www.nps.gov/yell/index.htm) (not in Utah, but much closer from here), lakes such as Mirror Lake and Silver Lake, and many outdoor trails. It’s an ideal place for a PhD student who wants to live close to nature and far from the hustle and bustle of a crowded city.


## 📸 Gallery
Here are photos I took during my visit to Yellowstone in Spring 2025:
[Yellowstone — Spring 2025](gallery.html)

-------------------------------------------------

## 🧘 Outside of Research

Yoga and meditation are how I reset. I practice regularly at CorePower Yoga, where Core Restore is my favorite class, and I love a good sound bath. They keep me grounded, and they remind me that clarity comes from stillness as much as from effort.

-------------------------------------------------

## 📖🎧 Recent favorite Books & Podcasts
- **[The Let Them Theory — Mel Robbins](https://www.melrobbins.com/book/the-let-them-theory/)**  
  _A practical lens on agency, focus, and action in complex work and life._

- **[The Diary Of A CEO](https://www.youtube.com/channel/UCGq-a57w-aPwyi3pW7XLiHw)**  
  _Long-form conversations about motivation, leadership, and personal evolution._

- **[Effortless — Greg McKeown](https://www.amazon.com/Effortless-Make-Easier-What-Matters/dp/B08FMTSZLN/)**  
  _A guide to making the most important work feel lighter, not harder._

- **[Being the Best (Behtarin Boodan) – Masoud Malekyari](https://ketabrah.com/book/60237)**  
  _A little flute loses its notes and creates a new song. Great work is not made from perfect conditions, but from turning limits into new strengths._

-------------------------------------------------

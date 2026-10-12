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
      // never behind the content. Each margin runs its own slow cycle:
      //   1. two dots light up, somewhere in the margin, near or far apart;
      //   2. a connection is tried from one toward the other, dot by dot. Often
      //      it reaches a dead end, pauses, and fades away, and another way is
      //      tried. This can happen up to three times; sometimes the first try
      //      already works;
      //   3. the try that reaches the other dot is kept. It glows gently and
      //      then dissolves, and after a short rest a new pair is chosen.
      // It starts once the status popup has been closed. The margins follow the
      // window size; on a narrow screen with no margins nothing is drawn.
      var canvas = document.getElementById('bgPattern');
      if (!canvas || !canvas.getContext) return;
      var ctx = canvas.getContext('2d');
      var GAP = 110, REACH = 210, MAX_FAILS = 3, QUICK = 0.3;
      var ENDS = 900, STEP = 480, STALL = 400, GIVE_UP = 650, SHINE = 1300, FADE = 1400;
      var still = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
      var w = 0, h = 0, pts = [], eps = [];
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
      function makeRoute(from, to, side) {
        // Wander from one end dot to the other in any direction, each hop
        // landing a little closer to the target.
        var cur = from, route = [from], used = {};
        used[from] = true;
        for (var guard = 0; guard < 40; guard++) {
          var left = gapBetween(cur, to), near = [];
          for (var j = 0; j < pts.length; j++) {
            if (j === to || used[j] || pts[j].side !== side) continue;
            if (gapBetween(cur, j) <= REACH && gapBetween(j, to) < left - GAP * 0.15) near.push(j);
          }
          var canFinish = left <= REACH * 1.15;
          if (canFinish && (!near.length || (route.length > 1 && Math.random() < 0.5))) { route.push(to); return route; }
          if (!near.length) return null;
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
        var routes = [], seen = {};
        for (var n = 0; n < 20 && routes.length < fails + 1; n++) {
          var route = makeRoute(from, to, side);
          if (!route || route.length < 3 || seen[route.join(',')]) continue;
          seen[route.join(',')] = true;
          routes.push(route);
        }
        if (!routes.length) return false;
        var tries = [], clock = ENDS;
        for (j = 0; j < routes.length; j++) {
          var last = j === routes.length - 1;
          var full = routes[j].length - 1;
          // A failed try stops at a dead end part of the way along.
          var hops = last ? full : 1 + Math.floor(Math.random() * (full - 1));
          var reached = clock + hops * STEP;
          tries.push({ route: routes[j], hops: hops, start: clock, reached: reached });
          clock = reached + STALL + GIVE_UP + 200;
        }
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
      function drawTry(tr, age, alpha, extra) {
        ctx.lineWidth = 1 + 0.5 * extra;
        ctx.strokeStyle = 'rgba(0,31,63,' + alpha + ')';
        for (var k = 0; k < tr.hops; k++) {
          var grow = 0.5 - 0.5 * Math.cos(3.1416 * clamp((age - tr.start - k * STEP) / STEP));
          if (grow <= 0) break;
          var A = pts[tr.route[k]], B = pts[tr.route[k + 1]];
          ctx.beginPath();
          ctx.moveTo(A.x, A.y);
          ctx.lineTo(A.x + (B.x - A.x) * grow, A.y + (B.y - A.y) * grow);
          ctx.stroke();
          if (grow > 0.99) dot(B, 2 + 0.6 * extra, alpha * 1.2);
        }
      }
      function draw(t) {
        ctx.clearRect(0, 0, w, h);
        for (var i = 0; i < pts.length; i++) dot(pts[i], 1.3, 0.08);
        ctx.lineCap = 'round';
        for (var e = eps.length - 1; e >= 0; e--) {
          var ep = eps[e], age = t - ep.t0;
          var win = ep.tries[ep.tries.length - 1];
          var shineStart = win.reached + 250;
          if (age > shineStart + SHINE + FADE) {
            nextAt[ep.side] = t + 900 + Math.random() * 1300;
            eps.splice(e, 1);
            continue;
          }
          var shine = age > shineStart ? Math.sin(3.1416 * clamp((age - shineStart) / SHINE)) : 0;
          var keep = 1 - clamp((age - shineStart - SHINE) / FADE);
          for (var r = 0; r < ep.tries.length - 1; r++) {
            var tr = ep.tries[r];
            var drop = 1 - clamp((age - tr.reached - STALL) / GIVE_UP);
            if (age >= tr.start && drop > 0) drawTry(tr, age, 0.15 * drop, 0);
          }
          if (age >= win.start) drawTry(win, age, (0.16 + 0.12 * shine) * keep, shine);
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

- **[Being the Best (persian book) – Masoud Malekyari](https://ketabrah.com/book/60237)**  
  _A little flute loses its notes and creates a new song. Great work is not made from perfect conditions, but from turning limits into new strengths._

-------------------------------------------------

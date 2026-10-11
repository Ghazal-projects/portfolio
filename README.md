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
      // never behind the content. Every three seconds a new pattern begins, the
      // left and right margins taking turns: a dot in the upper part and a dot
      // in the lower part light up, several different routes between them are
      // sketched faintly, dot by dot, from both ends, and once they have all
      // met the shortest route is picked. It shines while the others fade
      // away, and then it dissolves too. End dots and routes are new each time.
      // It starts once the status popup has been closed. The margins follow the
      // window size; on a narrow screen with no margins nothing is drawn.
      var canvas = document.getElementById('bgPattern');
      if (!canvas || !canvas.getContext) return;
      var ctx = canvas.getContext('2d');
      var GAP = 110, REACH = 210, ROUTES = 3;
      var EVERY = 3000, ENDS = 600, STEP = 260, DECIDE = 500, SHINE = 800, FADE = 900;
      var still = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
      var w = 0, h = 0, pts = [], eps = [];
      var lastFrame = 0, lastStart = -EVERY, side = 1;
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
      function makeRoute(from, to) {
        // Wander downward from one end dot to the other, leaning toward the target.
        var cur = from, route = [from];
        for (var guard = 0; guard < 40; guard++) {
          var near = [], canFinish = false;
          for (var j = 0; j < pts.length; j++) {
            if (pts[j].side !== side) continue;
            var dx = pts[j].x - pts[cur].x, dy = pts[j].y - pts[cur].y;
            var d2 = dx * dx + dy * dy;
            if (j === to) { canFinish = dy > 0 && d2 <= REACH * REACH * 1.3; continue; }
            if (dy < GAP * 0.4 || Math.abs(dx) > dy * 1.3 || d2 > REACH * REACH) continue;
            if (pts[j].y > pts[to].y - GAP * 0.3) continue;
            near.push(j);
          }
          if (canFinish && (!near.length || Math.random() < 0.5)) { route.push(to); return route; }
          if (!near.length) return null;
          var one = pick(near), two = pick(near);
          cur = Math.abs(pts[one].x - pts[to].x) <= Math.abs(pts[two].x - pts[to].x) ? one : two;
          route.push(cur);
        }
        return null;
      }
      function routeLength(route) {
        var total = 0;
        for (var i = 1; i < route.length; i++) {
          total += Math.sqrt(Math.pow(pts[route[i]].x - pts[route[i - 1]].x, 2) + Math.pow(pts[route[i]].y - pts[route[i - 1]].y, 2));
        }
        return total;
      }
      function startEpisode(t) {
        side = side === -1 ? 1 : -1;
        // The two end dots are picked afresh each time: one somewhere in the
        // upper part of the margin, one somewhere in the lower part.
        var j, tops = [], bottoms = [];
        for (j = 0; j < pts.length; j++) {
          if (pts[j].side !== side) continue;
          if (pts[j].y < h * 0.3) tops.push(j);
          else if (pts[j].y > h * 0.7) bottoms.push(j);
        }
        if (!tops.length || !bottoms.length) return false;
        var from = pick(tops), to = pick(bottoms);
        var routes = [], seen = {}, chosen = 0, half = 0;
        for (var tries = 0; tries < 14 && routes.length < ROUTES; tries++) {
          var route = makeRoute(from, to);
          if (!route || route.length < 3 || seen[route.join(',')]) continue;
          seen[route.join(',')] = true;
          routes.push(route);
        }
        if (!routes.length) return false;
        for (j = 0; j < routes.length; j++) {
          if (routeLength(routes[j]) < routeLength(routes[chosen])) chosen = j;
          half = Math.max(half, Math.ceil((routes[j].length - 1) / 2));
        }
        eps.push({ t0: t, from: from, to: to, routes: routes, chosen: chosen, half: half });
        return true;
      }
      function clamp(v) { return v < 0 ? 0 : (v > 1 ? 1 : v); }
      function dot(pt, radius, alpha) {
        ctx.fillStyle = 'rgba(0,31,63,' + alpha + ')';
        ctx.beginPath();
        ctx.arc(pt.x, pt.y, radius, 0, 6.2832);
        ctx.fill();
      }
      function drawRoute(route, age, alpha, extra) {
        // Segments are drawn from both ends at once, working toward the middle.
        var n = route.length - 1;
        ctx.lineWidth = 1 + 0.8 * extra;
        ctx.strokeStyle = 'rgba(0,31,63,' + alpha + ')';
        for (var k = 0; k < n; k++) {
          var fromStart = k <= n - 1 - k;
          var rank = fromStart ? k : n - 1 - k;
          var grow = 0.5 - 0.5 * Math.cos(3.1416 * clamp((age - ENDS - rank * STEP) / STEP));
          if (grow <= 0) continue;
          var A = pts[route[fromStart ? k : k + 1]], B = pts[route[fromStart ? k + 1 : k]];
          ctx.beginPath();
          ctx.moveTo(A.x, A.y);
          ctx.lineTo(A.x + (B.x - A.x) * grow, A.y + (B.y - A.y) * grow);
          ctx.stroke();
          if (grow > 0.99) dot(B, 2 + 0.9 * extra, alpha * 1.2);
        }
      }
      function draw(t) {
        ctx.clearRect(0, 0, w, h);
        for (var i = 0; i < pts.length; i++) dot(pts[i], 1.3, 0.08);
        ctx.lineCap = 'round';
        for (var e = eps.length - 1; e >= 0; e--) {
          var ep = eps[e], age = t - ep.t0;
          var joined = ENDS + ep.half * STEP;
          var shineStart = joined + DECIDE * 0.5;
          if (age > shineStart + SHINE + FADE) { eps.splice(e, 1); continue; }
          var others = 1 - clamp((age - joined) / DECIDE);
          var shine = age > shineStart ? Math.sin(3.1416 * clamp((age - shineStart) / SHINE)) : 0;
          var keep = 1 - clamp((age - shineStart - SHINE) / FADE);
          for (var r = 0; r < ep.routes.length; r++) {
            if (r === ep.chosen) continue;
            if (others > 0) drawRoute(ep.routes[r], age, 0.12 * others, 0);
          }
          drawRoute(ep.routes[ep.chosen], age, (0.14 + 0.36 * shine) * keep, shine);
          var appear = clamp(age / 400) * keep;
          dot(pts[ep.from], 3.2 + shine, (0.38 + 0.25 * shine) * appear);
          dot(pts[ep.to], 3.2 + shine, (0.38 + 0.25 * shine) * appear);
        }
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
          if (window.popupClosed && t - lastStart >= EVERY && startEpisode(t)) lastStart = t;
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

-------------------------------------------------

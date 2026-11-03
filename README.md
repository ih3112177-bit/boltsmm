
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>BoltSMM</title>
<style>
*{box-sizing:border-box}
body{margin:0;background:#10111b;color:white;font-family:Arial,sans-serif}
header{padding:20px;background:#191a29;font-size:24px;font-weight:bold}
header span{color:#8172ff}
main{max-width:650px;margin:35px auto;padding:20px}
.panel{background:#1c1d2d;padding:22px;border-radius:15px;margin-bottom:18px}
h1 span{color:#8172ff}
input,select{width:100%;padding:13px;margin:7px 0 14px;border-radius:8px;border:1px solid #444;background:#292a3b;color:white}
button{padding:12px 17px;border:0;border-radius:8px;background:#8172ff;color:white;font-weight:bold;cursor:pointer;margin:4px}
button:disabled{opacity:.5}
p{color:#bfc0d0;line-height:1.5}
.service,.order{padding:13px;border-bottom:1px solid #414252}
small{color:#aaa}
#message{color:#a99eff}
.hidden{display:none}
</style>
</head>
<body>
<header>⚡ Bolt<span>SMM</span></header>
<main>
<h1>Welcome to <span>BoltSMM</span></h1>
<p>Your social media marketing workspace.</p>

<section class="panel" id="auth">
<h2>Customer account</h2>
<input id="email" type="email" placeholder="Email address">
<input id="password" type="password" placeholder="Password">
<button id="signup">Create account</button>
<button id="login">Login</button>
<p id="message">Create an account or log in.</p>
</section>

<section class="panel hidden" id="workspace">
<h2>Your account</h2>
<p id="userEmail"></p>
<button id="logout">Log out</button>

<h2>Available services</h2>
<div id="services">Loading services...</div>

<h2>Place an order</h2>
<label>Service</label>
<select id="service"></select>

<label>Platform</label>
<select id="platform">
<option>Instagram</option>
<option>YouTube</option>
<option>Facebook</option>
<option>LinkedIn</option>
<option>TikTok</option>
<option>Other</option>
</select>

<label>Budget (₹)</label>
<input id="budget" type="number" min="100" placeholder="Minimum ₹100">

<label>Profile or website URL</label>
<input id="url" type="url" placeholder="https://...">

<label>Campaign brief</label>
<input id="brief" type="text" placeholder="Describe your requirements">

<button id="submitOrder">Submit order</button>

<h2>My orders</h2>
<div id="orders">No orders yet.</div>
</section>
</main>

<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script>
const SUPABASE_URL = "https://vgfdwjatydjulpbfnmqd.supabase.co/rest/v1/";
const SUPABASE_KEY = "sb_publishable_Kn4KpKIhk4l-wP0w4OY4bg_gpzTLLIh";

const db = window.supabase.createClient(
  SUPABASE_URL,
  SUPABASE_KEY
);

let serviceList = [];

const message = document.getElementById("message");

function showMessage(text) {
  message.textContent = text;
}

async function signup() {
  const email = document.getElementById("email").value.trim();
  const password = document.getElementById("password").value;

  if (!email || password.length < 8) {
    showMessage("Enter an email and password of at least 8 characters.");
    return;
  }

  const {error} = await db.auth.signUp({email,password});

  showMessage(error ? error.message :
    "Account created. Check your email if confirmation is required.");
}

async function login() {
  const email = document.getElementById("email").value.trim();
  const password = document.getElementById("password").value;

  const {error} = await db.auth.signInWithPassword({
    email,password
  });

  if (error) showMessage(error.message);
  else await loadWorkspace();
}

async function logout() {
  await db.auth.signOut();
  document.getElementById("workspace").classList.add("hidden");
  document.getElementById("auth").classList.remove("hidden");
  showMessage("You have logged out.");
}

async function loadServices() {
  const {data,error} = await db
    .from("services")
    .select("id,name,description,starting_price")
    .eq("active",true);

  if(error) {
    document.getElementById("services").textContent = error.message;
    return;
  }

  serviceList = data || [];

  document.getElementById("services").innerHTML =
    serviceList.map(s =>
      `<div class="service">
        <strong>${escapeHTML(s.name)}</strong>
        <p>${escapeHTML(s.description || "")}</p>
        <small>Starting at ₹${s.starting_price}</small>
      </div>`
    ).join("") || "No services available.";

  document.getElementById("service").innerHTML =
    serviceList.map(s =>
      `<option value="${escapeHTML(s.id)}">${escapeHTML(s.name)}</option>`
    ).join("");
}

function escapeHTML(value) {
  return String(value).replace(/[&<>"']/g, c => ({
    "&":"&amp;","<":"&lt;",">":"&gt;",
    '"':"&quot;","'":"&#39;"
  })[c]);
}

async function submitOrder() {
  const {data:{user}} = await db.auth.getUser();
  if(!user) return;

  const service_id = document.getElementById("service").value;
  const budget = Number(document.getElementById("budget").value);
  const url = document.getElementById("url").value.trim();
  const brief = document.getElementById("brief").value.trim();
  const platform = document.getElementById("platform").value;

  if(!service_id || !budget || budget < 100 || !url || !brief) {
    alert("Complete all fields. Minimum budget is ₹100.");
    return;
  }

  if(!/^https?:\/\//i.test(url)) {
    alert("Enter a valid URL beginning with https:// or http://");
    return;
  }

  const {error} = await db.from("orders").insert({
    user_id:user.id,
    service_id,
    platform,
    budget,
    url,
    brief
  });

  if(error) {
    alert("Order error: " + error.message);
    return;
  }

  alert("Order submitted successfully!");
  document.getElementById("budget").value="";
  document.getElementById("url").value="";
  document.getElementById("brief").value="";
  await loadOrders();
}

async function loadOrders() {
  const {data,error} = await db
    .from("orders")
    .select("id,service_id,platform,budget,status,created_at")
    .order("created_at",{ascending:false});

  const box = document.getElementById("orders");

  if(error) {
    box.textContent = error.message;
    return;
  }

  box.innerHTML = data.length ? data.map(o => {
    const service = serviceList.find(s => s.id === o.service_id);
    return `<div class="order">
      <strong>${escapeHTML(service?.name || o.service_id)}</strong>
      <p>${escapeHTML(o.platform)} · ₹${o.budget}</p>
      <small>Status: ${escapeHTML(o.status)}</small>
    </div>`;
  }).join("") : "No orders yet.";
}

async function loadWorkspace() {
  const {data:{user}} = await db.auth.getUser();
  if(!user) return;

  document.getElementById("auth").classList.add("hidden");
  document.getElementById("workspace").classList.remove("hidden");
  document.getElementById("userEmail").textContent = user.email;

  await loadServices();
  await loadOrders();
}

document.getElementById("signup").onclick = signup;
document.getElementById("login").onclick = login;
document.getElementById("logout").onclick = logout;
document.getElementById("submitOrder").onclick = submitOrder;

(async function start() {
  const {data:{session}} = await db.auth.getSession();
  if(session) await loadWorkspace();
})();
</script>
</body>
</html>

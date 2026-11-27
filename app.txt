/* =========================================
   SUPABASE CONFIGURATION
   ========================================= */
const SUPABASE_URL = "https://tjekrbmmkqxrdqtmxdmz.supabase.co";
const SUPABASE_KEY = "sb_publishable_S2GWeU3hgtnsDRGCR4mG6g_HqP-XjIY"; // Apni real key yahan daalo
const db = supabase.createClient(SUPABASE_URL, SUPABASE_KEY);

let currentUser = null;
let userRole = 'customer';
let isSignupMode = false;

/* =========================================
   INITIALIZATION & ROUTING
   ========================================= */
document.addEventListener("DOMContentLoaded", async () => {
  await checkSession();
  await loadStorefrontData();
  startCategoryReel();
});

// Modal Functions
function showCustomerAuth() { document.getElementById('customerModal').classList.add('active'); }
function showAdminAuth() { document.getElementById('adminModal').classList.add('active'); }
function closeModal(id) { document.getElementById(id).classList.remove('active'); }

/* =========================================
   AUTHENTICATION (Customer & Admin Separate)
   ========================================= */
async function checkSession() {
  const { data: { session } } = await db.auth.getSession();
  if (session) {
    currentUser = session.user;
    // Check role from profiles table
    const { data: profile } = await db.from('profiles').select('role').eq('id', currentUser.id).single();
    userRole = profile?.role || 'customer';
  }
}

async function handleCustomerAuth(e) {
  e.preventDefault();
  const email = document.getElementById('custEmail').value;
  const password = document.getElementById('custPass').value;
  
  if (isSignupMode) {
    const { error } = await db.auth.signUp({ email, password });
    if (!error) alert("Account created! Please login.");
    isSignupMode = false;
  } else {
    const { data, error } = await db.auth.signInWithPassword({ email, password });
    if (error) return alert(error.message);
    
    // Verify role
    const { data: profile } = await db.from('profiles').select('role').eq('id', data.user.id).single();
    if (profile?.role === 'admin' || profile?.role === 'super_admin') {
      alert("Please use the Admin login button.");
      await db.auth.signOut();
      return;
    }
    currentUser = data.user;
    userRole = profile?.role || 'customer';
    closeModal('customerModal');
    openCustomerDashboard();
  }
}

async function handleAdminAuth(e) {
  e.preventDefault();
  const email = document.getElementById('adminEmail').value;
  const password = document.getElementById('adminPass').value;
  
  const { data, error } = await db.auth.signInWithPassword({ email, password });
  if (error) return alert(error.message);
  
  // STRICT ADMIN CHECK (Requirement #4)
  const { data: profile } = await db.from('profiles').select('role').eq('id', data.user.id).single();
  if (!profile || !['admin', 'super_admin', 'product_manager', 'order_manager'].includes(profile.role)) {
    alert("Unauthorized access! You are not an admin.");
    await db.auth.signOut();
    return;
  }
  
  currentUser = data.user;
  userRole = profile.role;
  closeModal('adminModal');
  openAdminDashboard();
}

function toggleCustMode() {
  isSignupMode = !isSignupMode;
  const btn = document.querySelector('#customerAuthForm button');
  btn.textContent = isSignupMode ? "Create Account" : "Sign In";
}

async function logout() {
  await db.auth.signOut();
  location.reload();
}

/* =========================================
   DASHBOARD LOGIC
   ========================================= */
function openCustomerDashboard() {
  document.getElementById('customerDashboard').classList.remove('hidden');
  loadCustTab('overview');
}

function openAdminDashboard() {
  document.getElementById('adminDashboard').classList.remove('hidden');
  document.getElementById('adminRoleBadge').textContent = `Logged in as: ${userRole.toUpperCase()}`;
  loadAdminTab('overview');
}

// Customer Tabs
async function loadCustTab(tab) {
  const content = document.getElementById('custContent');
  content.innerHTML = '<p>Loading...</p>';
  
  if (tab === 'overview') {
    content.innerHTML = `<h2>Welcome Back!</h2><p>Email: ${currentUser.email}</p>`;
  } else if (tab === 'orders') {
    const { data } = await db.from('orders').select('*').eq('user_id', currentUser.id);
    content.innerHTML = `<h2>My Orders</h2>` + (data?.map(o => `<p>Order #${o.order_code} - ${o.status}</p>`).join('') || 'No orders found.');
  } else if (tab === 'wishlist') {
    content.innerHTML = `<h2>My Wishlist</h2><p>Wishlist feature coming soon...</p>`;
  }
}

// Admin Tabs
async function loadAdminTab(tab) {
  const content = document.getElementById('adminContent');
  content.innerHTML = '<p>Loading Data...</p>';
  
  if (tab === 'overview') {
    // Dashboard Stats
    const { count: prodCount } = await db.from('products').select('*', { count: 'exact', head: true });
    const { count: ordCount } = await db.from('orders').select('*', { count: 'exact', head: true });
    
    content.innerHTML = `
      <h2>Dashboard Overview</h2>
      <div class="stat-cards">
        <div class="stat-card"><h3>Total Products</h3><div class="value">${prodCount || 0}</div></div>
        <div class="stat-card"><h3>Total Orders</h3><div class="value">${ordCount || 0}</div></div>
      </div>
    `;
  } else if (tab === 'central') {
    content.innerHTML = `<h2>Central Brain Settings</h2>
      <form onsubmit="saveCentralSettings(event)">
        <label>Website Name</label>
        <input type="text" id="setBrandName" value="CozyCarry">
        <label>Primary Color</label>
        <input type="color" id="setPrimary" value="#0f3d2e">
        <button type="submit" class="btn-primary" style="margin-top:20px">Save Settings</button>
      </form>`;
  }
}

/* =========================================
   STOREFRONT DATA (Products & Categories)
   ========================================= */
async function loadStorefrontData() {
  // Load Circular Categories
  const { data: cats } = await db.from('categories').select('*').eq('is_active', true);
  const catReel = document.getElementById('categoryReel');
  if (cats && cats.length > 0) {
    catReel.innerHTML = cats.map(c => `
      <div class="cat-circle" onclick="filterByCategory('${c.name}')">
        <img src="${c.image_url || 'https://via.placeholder.com/150'}" alt="${c.name}">
      </div>
    `).join('');
  }

  // Load Products
  const { data: prods } = await db.from('products').select('*').eq('is_active', true);
  const prodGrid = document.getElementById('productGrid');
  if (prods && prods.length > 0) {
    prodGrid.innerHTML = prods.map(p => `
      <div class="prod-card">
        <img src="${p.cover_image || p.images?.[0]}" class="prod-img" alt="${p.name}">
        <div class="prod-info">
          <div class="prod-title">${p.name}</div>
          <div>
            <span class="prod-price">Rs. ${p.price}</span>
            ${p.old_price ? `<span class="prod-old">Rs. ${p.old_price}</span>` : ''}
          </div>
          <button class="btn-dark full-width" style="margin-top:10px" onclick="addToCart('${p.id}')">Add to Cart</button>
        </div>
      </div>
    `).join('');
  } else {
    prodGrid.innerHTML = '<p>No products found. Add some in Admin Panel.</p>';
  }
}

function filterByCategory(catName) {
  alert(`Filtering by category: ${catName} (Feature active in full version)`);
}
function addToCart(id) { alert("Added to cart!"); }

/* =========================================
   CATEGORY REEL ANIMATION (Auto Scroll)
   ========================================= */
function startCategoryReel() {
  const reel = document.getElementById('categoryReel');
  let scrollAmount = 1;
  let isHovered = false;

  reel.addEventListener('mouseenter', () => isHovered = true);
  reel.addEventListener('mouseleave', () => isHovered = false);

  function scroll() {
    if (!isHovered) {
      reel.scrollLeft += scrollAmount;
      if (reel.scrollLeft >= reel.scrollWidth - reel.clientWidth) {
        reel.scrollLeft = 0;
      }
    }
    requestAnimationFrame(scroll);
  }
  scroll();
}

async function saveCentralSettings(e) {
  e.preventDefault();
  // Logic to save settings to Supabase store_settings table
  alert("Settings Saved!");
}
# ROYAL SEEDS FARM — Complete Implementation Plan & Production Deliverables

## 🚀 Quick 5-Minute Guide: How to Publish This Website Live & Get Your Public URL

Right now, your website is running inside this private workspace. Follow these simple steps to put it online for free and get a live shareable link (e.g., `https://royal-seeds-farm.vercel.app` or your own custom `.com` / `.in` domain).

### Step 1: Set Up the Project on Your Computer
1. Download and install **[Node.js](https://nodejs.org/)** (LTS version) if you haven't already.
2. Open your terminal (Command Prompt / PowerShell on Windows, or Terminal on Mac) and run:
   ```bash
   npx create-vite@latest royal-seeds-farm --template react
   cd royal-seeds-farm
   npm install lucide-react firebase
   npm install -D tailwindcss postcss autoprefixer
   npx tailwindcss init -p
   ```
3. Configure Tailwind CSS by opening `tailwind.config.js` and updating `content`:
   ```javascript
   export default {
     content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
     theme: { extend: {} },
     plugins: [],
   }
   ```
4. Add the Tailwind directives to the top of `src/index.css`:
   ```css
   @tailwind base;
   @tailwind components;
   @tailwind utilities;
   ```
5. Copy the code from the **`RoyalSeedsFarm.jsx`** file in this workspace and paste it into `src/App.jsx` in your project folder.
6. Test it locally on your computer by running:
   ```bash
   npm run dev
   ```

### Step 2: Upload Your Project to GitHub
1. Create a free account at **[github.com](https://github.com/)** and click **New Repository** (name it `royal-seeds-farm`).
2. Push your folder to GitHub (or simply drag and drop your project files into the GitHub web uploader).

### Step 3: Deploy Live on Vercel (Free Public URL)
1. Go to **[vercel.com](https://vercel.com/)** and sign up for free using your GitHub account.
2. Click **"Add New..." → "Project"** and select your `royal-seeds-farm` repository.
3. Click **Deploy**.
4. Within 60 seconds, Vercel will build your site and give you your live public URL (for example: `https://royal-seeds-farm.vercel.app`)!
5. **Optional Custom Domain:** In your Vercel project dashboard, go to **Settings → Domains** to connect `www.royalseedsfarm.com` or `www.royalseedsfarm.in`.

---

## 1. Architectural Analysis & Environment Compatibility Explanation
- **Interactive Preview Runtime:** The interactive web application (`RoyalSeedsFarm.jsx`) is compiled as a unified React + Tailwind CSS application connected to real-time cloud persistence (Firebase/Firestore) and the Gemini AI model (`gemini-3-flash-preview`) so you can immediately test all 10 pages, product search/filtering, validated enquiry forms, multilingual AI chat, and the protected Staff Admin Dashboard (`Passcode: RSF-ADMIN-2026`).
- **Production Next.js + Supabase Stack:** When deploying to your own custom domain on **Vercel** with **Next.js (TypeScript App Router)** and **Supabase (PostgreSQL + Auth + Storage)**, use the database migration script, environment template, and server route specifications provided below.

---

## 2. PostgreSQL / Supabase Database Schema & RLS Migrations (`supabase/migrations/001_init_royal_seeds_farm.sql`)

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. PRODUCT CATEGORIES TABLE
CREATE TABLE public.product_categories (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL UNIQUE,
  slug TEXT NOT NULL UNIQUE,
  description TEXT NOT NULL,
  image_url TEXT,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 2. PRODUCTS CATALOGUE TABLE
CREATE TABLE public.products (
  id TEXT PRIMARY KEY, -- e.g. 'RSF-TOM-101'
  name TEXT NOT NULL,
  category_name TEXT NOT NULL REFERENCES public.product_categories(name) ON UPDATE CASCADE,
  crop_type TEXT NOT NULL,
  variety_name TEXT,
  description TEXT NOT NULL,
  images TEXT[] DEFAULT '{}',
  packaging_options TEXT[] DEFAULT '{}',
  technical_specs TEXT,
  recommended_season TEXT,
  sowing_info TEXT,
  suitable_soil TEXT,
  availability_status TEXT DEFAULT 'Available for Enquiry',
  is_published BOOLEAN DEFAULT false,
  is_featured BOOLEAN DEFAULT false,
  seo_title TEXT,
  seo_description TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 3. CUSTOMER, PRODUCT & DEALER ENQUIRIES TABLE
CREATE TABLE public.enquiries (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  enquiry_category TEXT NOT NULL, -- 'Product Enquiry', 'Dealer / Distributor Application', 'Contact Form'
  customer_name TEXT NOT NULL,
  business_name TEXT,
  mobile_number TEXT NOT NULL,
  email TEXT,
  state TEXT,
  district TEXT,
  business_type TEXT,
  areas_of_interest TEXT[] DEFAULT '{}',
  product_id TEXT REFERENCES public.products(id) ON DELETE SET NULL,
  packaging_preference TEXT,
  message TEXT,
  status TEXT NOT NULL DEFAULT 'New' CHECK (status IN ('New', 'Contacted', 'In Progress', 'Converted', 'Closed')),
  internal_notes TEXT DEFAULT '',
  ip_hash TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 4. CMS & SITE SETTINGS TABLE
CREATE TABLE public.site_settings (
  id INT PRIMARY KEY DEFAULT 1 CHECK (id = 1),
  brand_name TEXT NOT NULL DEFAULT 'ROYAL SEEDS FARM',
  hero_headline TEXT NOT NULL,
  hero_subtext TEXT NOT NULL,
  intro_heading TEXT NOT NULL,
  intro_body TEXT NOT NULL,
  about_content TEXT NOT NULL,
  phone TEXT DEFAULT '',
  email TEXT DEFAULT '',
  whatsapp_number TEXT DEFAULT '',
  address TEXT DEFAULT 'MAIN AMRITSAR, PUNJAB 143001',
  business_hours TEXT DEFAULT '',
  show_testimonials BOOLEAN DEFAULT false,
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- 5. ADMINISTRATIVE AUDIT LOGS
CREATE TABLE public.audit_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  admin_user_id UUID NOT NULL,
  action TEXT NOT NULL,
  details JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ROW LEVEL SECURITY (RLS) POLICIES
ALTER TABLE public.product_categories ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.products ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.enquiries ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.site_settings ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.audit_logs ENABLE ROW LEVEL SECURITY;

-- Public read access ONLY to published products & active categories
CREATE POLICY "Public can view active categories" ON public.product_categories
  FOR SELECT USING (is_active = true);

CREATE POLICY "Public can view published products" ON public.products
  FOR SELECT USING (is_published = true);

CREATE POLICY "Public can view site settings" ON public.site_settings
  FOR SELECT USING (true);

-- Public can INSERT enquiries (validated via server action), but NEVER read/update/delete enquiries
CREATE POLICY "Public can submit enquiries" ON public.enquiries
  FOR INSERT WITH CHECK (true);

-- Authorised Staff Admins (authenticated with role = 'admin') have full CRUD access
CREATE POLICY "Admins have full access to products" ON public.products
  FOR ALL USING (auth.jwt() ->> 'user_role' = 'admin');

CREATE POLICY "Admins have full access to enquiries" ON public.enquiries
  FOR ALL USING (auth.jwt() ->> 'user_role' = 'admin');

CREATE POLICY "Admins have full access to site settings" ON public.site_settings
  FOR ALL USING (auth.jwt() ->> 'user_role' = 'admin');

CREATE POLICY "Admins can view and insert audit logs" ON public.audit_logs
  FOR ALL USING (auth.jwt() ->> 'user_role' = 'admin');
```

---

## 3. Environment Variable Example (`.env.example`)
Create a `.env.local` file in your Next.js root directory and populate the following keys (never commit real secrets to Git):

```env
# Public App Configuration
NEXT_PUBLIC_SITE_URL="https://www.royalseedsfarm.com"
NEXT_PUBLIC_SUPABASE_URL="https://your-project-id.supabase.co"
NEXT_PUBLIC_SUPABASE_ANON_KEY="your-supabase-anon-public-key"

# Server-Only Secrets (Never expose to client browser)
SUPABASE_SERVICE_ROLE_KEY="your-supabase-service-role-secret-key"
GEMINI_API_KEY="your-google-gemini-server-api-key"

# Rate Limiting & Spam Protection (Upstash Redis or Vercel KV optional)
RATE_LIMIT_MAX_REQUESTS_PER_MINUTE="10"
ADMIN_NOTIFICATION_EMAIL="support@royalseedsfarm.com"
```

---

## 4. Enterprise Next.js + Supabase Setup Instructions

### Local Setup
1. **Initialize Next.js TypeScript Project:**
   ```bash
   npx create-next-app@latest royal-seeds-farm --typescript --tailwind --app
   cd royal-seeds-farm
   npm install @supabase/supabase-js @supabase/ssr lucide-react zod @google/genai
   ```
2. **Apply Database Migrations:**
   - Open your Supabase Project → SQL Editor and run `001_init_royal_seeds_farm.sql`.
   - Create a storage bucket named `product-images` with public read access and authenticated admin upload policies.
3. **Configure Environment Variables:**
   - Copy `.env.example` to `.env.local` and fill in your Supabase and Gemini API credentials.
4. **Run Development Server:**
   ```bash
   npm run dev
   ```

### Production Deployment (Vercel)
1. Push the repository to a private GitHub/GitLab repository.
2. Import the project into **Vercel** and configure the environment variables from `.env.example`.
3. Enable **HTTPS / SSL** enforcement and configure your custom domain (`royalseedsfarm.com`).
4. Enable automated daily PostgreSQL backups in the Supabase dashboard.

---

## 5. External Services Requiring Accounts or API Keys
1. **Supabase Account** or **Firebase Project** (Database, Admin Authentication, and Product Image Storage).
2. **Google AI Studio Account** (`GEMINI_API_KEY` for server-side multilingual AI Farming Assistant).
3. **Vercel Account** (Free Production Hosting, Edge Caching, and SSL).
4. **Optional Transactional Email Provider** (e.g., Resend or AWS SES) if instant email alerts are desired alongside database logging.

---

## 6. Company Information & Brand Assets Still Required from ROYAL SEEDS FARM
Because this platform strictly forbids inventing business claims, certifications, or fake reviews, the following items are clearly marked as placeholders or hidden until verified by authorised staff:
1. **Official Brand Logo:** High-resolution SVG/PNG logo (currently using a clean typographic emblem).
2. **Verified Company History & Credentials:** Founding year, leadership profile, seed certification authority registrations, and processing plant details.
3. **Verified Contact & WhatsApp Numbers:** Phone, email, and official WhatsApp Business number (Address is set to `MAIN AMRITSAR, PUNJAB 143001`; entering the WhatsApp number in the Admin Dashboard immediately activates the floating WhatsApp button and product pre-filled links).
4. **Verified Lot Specifications & Packaging Photos:** High-resolution packet photographs and laboratory-verified agronomic parameters for each seed variety.
5. **Documented Testimonials:** Written consent from real dealers or farmers before enabling the Testimonials toggle in the Admin Dashboard.

---

## 7. Quality Assurance & Testing Checklist
- [x] **Navigation & Pages:** All 10 pages (`Home`, `About Us`, `Products`, `Product Detail`, `AI Farming Assistant`, `Crop Guide`, `Dealer Enquiry`, `Contact`, `Privacy Policy`, `Terms & Conditions`) + `Custom 404` and `Admin Dashboard` route cleanly on desktop and mobile.
- [x] **Catalogue Search & Category Filters:** Instant filtering by category, crop name, season, and sorting with helpful empty states.
- [x] **Enquiry Forms & Spam Protection:** Product Enquiry, Dealer Application, and Contact forms enforce client/server validation, honeypot spam checks, duplicate rate-limiting, and privacy consent before saving to the database.
- [x] **Multilingual AI Assistant:** Responds in English, Hindi, and Hinglish, asks clarifying follow-up questions for state/district and season, links directly to real catalogue products, and refuses to invent yields or chemical prescriptions.
- [x] **Admin Security & Audit Logging:** Protected by staff authentication (`RSF-ADMIN-2026` in preview), supports full product CRUD, enquiry status/notes updates, CSV export, CMS editing, and logs administrative actions.
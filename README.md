
## Todo List

### I. Project Setup & Initial Configuration

1.  [ ] **Clerk Project Setup:**
    *   [ ] Create a Clerk account: [https://clerk.com/](https://clerk.com/)
    *   [ ] Create a new Clerk application.
    *   [ ] Configure redirect URLs in the Clerk dashboard (e.g., `http://localhost:3000`, your production domain).
    *   [ ] Retrieve your Clerk Publishable Key and Secret Key (store securely as environment variables: `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY`).
2.  [ ] **Supabase Project Setup:**
    *   [ ] Create a Supabase account: [https://supabase.com/](https://supabase.com/)
    *   [ ] Create a new Supabase project.
    *   [ ] Retrieve your Supabase URL and Service Role Key (store securely as environment variables: `NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`).  **Important:** Never expose your `SUPABASE_SERVICE_ROLE_KEY` to the client! Only use it in server-side code.
3.  [ ] **Next.js Project Setup:**
    *   [ ] Create a new Next.js project: `npx create-next-app@latest my-user-management-template` (Choose App Router, TypeScript, ESLint, Tailwind CSS, `src/` directory, import alias `@/*`).
    *   [ ] Navigate to the project directory: `cd my-user-management-template`

### II. Dependency Installation

1.  [ ] **Install Core Dependencies:**

    ```bash
    npm install @clerk/nextjs @supabase/supabase-js drizzle-orm drizzle-kit postgres pg pg-protocol tailwindcss postcss autoprefixer class-variance-authority clsx tailwind-merge lucide-react @radix-ui/react-slot
    ```

2.  [ ] **Install Development Dependencies:**

    ```bash
    npm install -D typescript @types/node @types/react @types/react-dom eslint eslint-config-next prettier eslint-config-prettier eslint-plugin-tailwindcss
    ```

3.  [ ] **Install ShadCN UI dependencies**

    ```bash
    npx shadcn-ui@latest init
    # Use default answers and styles (New York)
    ```

### III. Database Schema and Policies (Supabase)

1.  [ ] **Connect to Supabase:** Connect to Supabase via the web UI.
2.  [ ] **Update `users` Table:**

    ```sql
    -- Already exists, but adding here for clarity if you hadn't run the prev README

    CREATE TABLE users (
        id SERIAL PRIMARY KEY,
        clerk_id TEXT UNIQUE NOT NULL,
        email TEXT NOT NULL,
        first_name TEXT,
        last_name TEXT,
        role TEXT NOT NULL DEFAULT 'user',  -- "user", "admin", "manager", "staff"
        profile_picture_url TEXT,        -- URL to the profile picture in Supabase Storage
        created_at TIMESTAMPTZ DEFAULT NOW()
    );

    -- Optional: Create an index on the clerk_id for faster lookups
    CREATE INDEX idx_users_clerk_id ON users (clerk_id);
    ```

3.  [ ] **Create `roles` Table:** (NEW)

    ```sql
    CREATE TABLE roles (
        id SERIAL PRIMARY KEY,
        name TEXT UNIQUE NOT NULL,  -- e.g., "admin", "manager", "staff"
        description TEXT,
        permissions JSONB  -- Store permissions as a JSON object
    );

    -- Initial roles
    INSERT INTO roles (name, description, permissions) VALUES
    ('user', 'Default user role', '{}'),
    ('admin', 'Administrator role', '{"all": true}'), -- Or specific permissions
    ('manager', 'Manager role', '{"manage_users": true, "view_reports": true}'),
    ('staff', 'Staff role', '{"create_content": true}');
    ```

4.  [ ] **Update `drizzle/schema.ts`**

    ```typescript
    import { pgTable, serial, text, timestamp, pgEnum, jsonb } from 'drizzle-orm/pg-core';

    export const userRoleEnum = pgEnum('user_role', ['user', 'admin', 'manager', 'staff']);

    export const users = pgTable('users', {
        id: serial('id').primaryKey(),
        clerkId: text('clerk_id').notNull().unique(),
        email: text('email').notNull(),
        firstName: text('first_name'),
        lastName: text('last_name'),
        role: userRoleEnum('role').notNull().default('user'),
        profilePictureUrl: text('profile_picture_url'),
        createdAt: timestamp('created_at').defaultNow(),
    });

    export const roles = pgTable('roles', { // NEW
      id: serial('id').primaryKey(),
      name: text('name').notNull().unique(),
      description: text('description'),
      permissions: jsonb('permissions'),
    });
    ```

5.  [ ] **Enable Row-Level Security (RLS) on `users`:**
    *   [ ] In the Supabase UI, navigate to the `users` table and go to "Policies."
    *   [ ] Enable RLS.
    *   [ ] Add the following policies (adjust as needed based on your specific access control requirements):

        *   **Policy: `Enable read access for all users`**
            *   Name: `Enable read access for all users`
            *   Target roles: `Authenticated`
            *   Using expression: `true` -- All authenticated users can see all user data.  Consider restricting this further if needed.

        *   **Policy: `Enable insert for new users`**
            *   Name: `Enable insert for new users`
            *   Target roles: `Authenticated`
            *   With check: `auth.uid() = clerk_id` --  Only the user can create their own record (Clerk webhook will handle this).

        *   **Policy: `Enable update for own record`**
            *   Name: `Enable update for own record`
            *   Target roles: `Authenticated`
            *   Using expression: `auth.uid() = clerk_id` -- Users can only see their OWN data.
            *   With check: `auth.uid() = clerk_id` -- Users can only update their OWN data.

        *   **Policy: `Enable admin read access`**
            *   Name: `Enable admin read access`
            *   Target roles: `Authenticated`
            *   Using expression: `EXISTS ( SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin' )` -- Only admins can see all user data

        *   **Policy: `Enable admin update access`**
            *   Name: `Enable admin update access`
            *   Target roles: `Authenticated`
            *   Using expression: `EXISTS ( SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin' )` -- Only admins can see all user data
            *   With Check: `role IN ('user', 'staff', 'manager')` -- Restrict an Admin from modifying the admin role

        *   **Policy: `Enable admin delete access`**
            *   Name: `Enable admin delete access`
            *   Target roles: `Authenticated`
            *   Using expression: `EXISTS ( SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin' )` -- Only admins can see all user data

            **Important Considerations for RLS:**

            *   **`auth.uid()` in Supabase:**  This corresponds to the user's UUID in Supabase *after* you've synced the Clerk user to Supabase.  That is the reason we store the clerk_id.
            *   **Carefully consider the implications of each policy.**  These are just examples. Adjust them to meet your specific security requirements.
            *   **Test your RLS policies thoroughly.**

        *   **RLS on `roles` table:** (NEW)
            *   Admin Only access to create, read, update, delete `roles` table.

6.  [ ] **Supabase Storage Bucket for Profile Pictures:**

    *   [ ] In the Supabase UI, navigate to "Storage."
    *   [ ] Create a new bucket (e.g., `user-profile-pictures`).
    *   [ ] Set the bucket to public or private (depending on your needs). Public is simpler, but private is more secure.
    *   [ ] If the bucket is private, you'll need to generate signed URLs to access the images.

### IV. Clerk Webhook Implementation

1.  [ ] **Create `app/api/webhooks/clerk/route.ts`:**

    ```typescript
    import { WebhookEvent } from '@clerk/clerk-sdk-node';
    import { headers } from 'next/headers';
    import { Webhook } from 'svix';
    import { db } from '@/db/db';
    import { users } from '@/db/schema';

    const webhookSecret = process.env.CLERK_WEBHOOK_SECRET || '';

    export async function POST(req: Request) {
      const payload = await req.text();
      const headersList = headers();
      const svixId = headersList.get('svix-id');
      const svixTimestamp = headersList.get('svix-timestamp');
      const svixSignature = headersList.get('svix-signature');
      if (!svixId || !svixTimestamp || !svixSignature) {
        return new Response('Error occured -- no svix headers', {
          status: 400,
        });
      }
      const svix = new Webhook(webhookSecret);
      let evt: WebhookEvent;
      try {
        evt = svix.verify(payload, {
          'svix-id': svixId,
          'svix-timestamp': svixTimestamp,
          'svix-signature': svixSignature,
        }) as WebhookEvent;
      } catch (err) {
        console.error('Error verifying webhook signature', err);
        return new Response('Error occured -- invalid svix signature', {
          status: 400,
        });
      }
      const { data, type } = evt;

      switch (type) {
        case 'user.created':
          try {
            await db.insert(users).values({
              clerkId: data.id,
              email: data.email_addresses[0].email_address,
              firstName: data.first_name,
              lastName: data.last_name,
            });
            console.log('User created in Supabase:', data.id);
          } catch (error) {
            console.error('Error creating user in Supabase:', error);
          }
          break;
        // Add other cases for user.updated, user.deleted, etc. as needed
        case 'user.updated':
          try {
            await db
              .update(users)
              .set({
                email: data.email_addresses[0].email_address,
                firstName: data.first_name,
                lastName: data.last_name,
                profilePictureUrl: data.image_url,
              })
              .where({ clerkId: data.id });

            console.log('User updated in Supabase:', data.id);
          } catch (error) {
            console.error('Error updating user in Supabase:', error);
          }
          break;
        case 'user.deleted':
          try {
            await db.delete(users).where({ clerkId: data.id });
            console.log('User deleted in Supabase:', data.id);
          } catch (error) {
            console.error('Error deleting user in Supabase:', error);
          }
          break;
      }

      return new Response('', { status: 200 });
    }
    ```

2.  [ ] **Set up Clerk Webhook:**
    *   [ ] In your Clerk dashboard, navigate to "Webhooks."
    *   [ ] Add a new webhook endpoint.
    *   [ ] Set the URL to `https://your-domain.com/api/webhooks/clerk` (replace with your actual domain).  For local development, use a tool like `ngrok` to expose your local server to the internet.
    *   [ ] Select the events you want to subscribe to (at least `user.created`, `user.updated`, `user.deleted`).
    *   [ ]  **Clerk Webhook Secret:** Clerk will generate a signing secret (`CLERK_WEBHOOK_SECRET`). Store this securely as an environment variable.  This secret is used to verify that the webhook requests are coming from Clerk and not from a malicious actor.

### V.  Drizzle Configuration

1.  [ ] **Create `drizzle.config.ts`:**

    ```typescript
    import type { Config } from 'drizzle-kit';
    import * as dotenv from 'dotenv';
    dotenv.config();

    if (!process.env.DATABASE_URL) {
        throw new Error('DATABASE_URL is not defined');
    }

    export default {
        schema: './db/schema.ts',
        out: './drizzle',
        driver: 'pg',
        dbCredentials: {
            connectionString: process.env.DATABASE_URL || '',
        },
        tablesFilter: ['users','roles'],
    } satisfies Config;
    ```

2.  [ ] **Run Drizzle Migrations:**

    ```bash
    drizzle-kit generate:pg --config=drizzle.config.ts  # Generate migrations
    drizzle-kit push:pg --config=drizzle.config.ts      # Push migrations to Supabase
    ```

3. [ ] **Introspect:**

    ```bash
    drizzle-kit introspect:pg --config=drizzle.config.ts
    ```

### VI.  UI Implementation (Tailwind CSS, ShadCN)

1.  [ ] **Basic Layout (`app/layout.tsx`):**

    ```typescript
    import { ClerkProvider } from '@clerk/nextjs';
    import './globals.css'; // Import your global CSS
    import { cn } from "@/lib/utils"
    import { Inter } from "next/font/google"

    const inter = Inter({
      subsets: ["latin"],
      variable: "--font-sans",
    })

    export const metadata = {
      title: "User Management App",
      description: "Template with Clerk Auth, Supabase, Drizzle ORM, and ShadCN",
    }

    export default function RootLayout({ children }: { children: React.ReactNode }) {
      return (
        <ClerkProvider>
          <html lang="en" suppressHydrationWarning>
            <body className={cn(
              "min-h-screen bg-background font-sans antialiased",
              inter.variable
            )}>
                {children}
            </body>
          </html>
        </ClerkProvider>
      );
    }
    ```

2.  [ ] **Dashboard (`app/dashboard/page.tsx`):**

    *   [ ] Fetch user role from Supabase.
    *   [ ] Conditionally render content based on the user's role.
    *   [ ] Include a link to the "Edit Profile" page.

    ```typescript
    import { auth } from "@clerk/nextjs";
    import { redirect } from "next/navigation";
    import { db } from "@/db/db";
    import { users } from "@/db/schema";
    import { eq } from "drizzle-orm";

    async function getProfile() {
      const { userId } = auth();

      if (!userId) {
        return redirect("/sign-in");
      }

      const user = await db.select().from(users).where(eq(users.clerkId, userId));
      return user[0];
    }

    export default async function Dashboard() {
      const user = await getProfile();

      return (
        <div>
          <h1>Dashboard</h1>
          <p>Welcome, {user?.firstName}!</p>
          <p>Your role: {user?.role}</p>
          <a href="/dashboard/edit-profile">Edit Profile</a>

          {user?.role === 'admin' && (
            <div>
              <h2>Admin Panel</h2>
              {/* Admin-only features */}
              <a href="/users">Manage Users</a>
              <a href="/admin/roles">Manage Roles</a>  {/* NEW */}
            </div>
          )}

          {user?.role === 'manager' && (
            <div>
              <h2>Manager Panel</h2>
              {/* Manager-only features */}
            </div>
          )}

          {user?.role === 'staff' && (
            <div>
              <h2>Staff Panel</h2>
              {/* Staff-only features */}
            </div>
          )}
        </div>
      );
    }
    ```

3.  [ ] **Edit Profile Page (`app/dashboard/edit-profile/page.tsx`):**

    *   [ ] Create a form with fields for updating user profile information (e.g., first name, last name, profile picture).
    *   [ ] Use a Server Action to handle the form submission.
    *   [ ] In the Server Action:
        *   [ ] Authenticate the user.
        *   [ ] Authorize the user (ensure they can only edit their own profile).
        *   [ ] Validate the input data.
        *   [ ] Update the user's record in the Supabase database.
        *   [ ] If the user uploads a new profile picture:
            *   [ ] Upload the image to Supabase Storage.
            *   [ ] Store the URL of the image in the `profile_picture_url` column of the `users` table.
        *   [ ] Revalidate the route.

4.  [ ] **User List Page (`app/users/page.tsx`):**
    *   [ ] Restrict access to administrators only.
    *   [ ] Display a list of all users, with links to edit individual users.

5.  [ ] **Edit User Page (`app/users/[id]/edit/page.tsx`):**
    *   [ ] Restrict access to administrators only.
    *   [ ] Create a form for editing user information (including role).
    *   [ ] Use a Server Action to handle the form submission.

6.  [ ] **Roles Page (`app/admin/roles/page.tsx`):**  (NEW)

    *   [ ] Restrict access to administrators only.
    *   [ ] Display a list of roles with options to create, edit, and delete them.

7.  [ ] **Role Server Actions (`app/admin/roles/actions.ts`):**  (NEW)

    *   [ ] Implement Server Actions for creating, updating, and deleting roles.

### VII. Environment Variables

1.  [ ] Create a `.env.local` file in your project root.
2.  [ ] Add the following environment variables:

    ```
    NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=your_clerk_publishable_key
    CLERK_SECRET_KEY=your_clerk_secret_key
    NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
    NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key  #Public
    SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_key #Private - only on the server
    DATABASE_URL=your_supabase_database_url
    CLERK_WEBHOOK_SECRET=your_clerk_webhook_secret
    ```

### VIII. Testing & Deployment

1.  [ ] **Testing:**
    *   [ ] Write unit tests for your Server Actions and components.
    *   [ ] Write integration tests to verify the interactions between Clerk, Supabase, and Drizzle.
    *   [ ] Write end-to-end tests to test the entire user flow.
2.  [ ] **Deployment:**
    *   [ ] Choose a hosting provider (e.g., Vercel, Netlify).
    *   [ ] Configure environment variables in your hosting provider's dashboard.
    *   [ ] Deploy your application.

This comprehensive README should give you a solid starting point for building your user management template!
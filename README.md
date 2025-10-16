# ConnectAI 🚀

A Streamlit web application that helps you find and track job opportunities by scraping LinkedIn and providing intelligent notifications.
You can check the website in this link : https://connect-ai.streamlit.app 

## Features

- 🔍 **LinkedIn Job Scraping**: Automatically search for jobs on LinkedIn
- 📊 **Dashboard**: Track and manage your job applications
- 🔄 **Continuous Search**: Background monitoring for new opportunities
- 📱 **Telegram Notifications**: Get notified instantly when new jobs are found
- 📝 **Application Tracking**: Keep track of application status and notes
- 👥 **User Management**: Multi-user support with secure authentication

## Setup Instructions

### Prerequisites

- Python 3.8+
- A Supabase account (for database)
- A Telegram bot (optional, for notifications)

### Installation

1. **Clone the repository**
   ```bash
   git clone https://github.com/mohamedazizhadjkacem/ConnectAI
   cd ConnectAI
   ```

2. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

3. **Set up environment variables**
   ```bash
   # Copy the template secrets file
   copy .streamlit\secrets.toml.template .streamlit\secrets.toml
   ```

4. **Configure your secrets**
   
   Edit `.streamlit/secrets.toml` with your actual values:
   ```toml
   [SUPABASE]
   SUPABASE_URL = "https://your-project.supabase.co"
   SUPABASE_KEY = "your-supabase-anon-key-here"
   ```

5. **Set up Supabase Database**

    The application uses the following tables in Supabase: `profiles`, `subscriptions`, and `internships`.
    Below are idempotent SQL snippets you can run in the Supabase SQL editor to create the expected schema (they use `IF NOT EXISTS` where available so you can safely re-run them).

    ```sql
    -- Profiles table (one row per authenticated user, id should match auth user id)
    CREATE TABLE IF NOT EXISTS public.profiles (
       id uuid PRIMARY KEY,
       username varchar,
       telegram_bot_token varchar,
       telegram_chat_id varchar,
       resume jsonb,
       resume_updated_at timestamptz,
       created_at timestamptz DEFAULT now()
    );

    -- Subscriptions table (per-user subscription metadata)
    CREATE TABLE IF NOT EXISTS public.subscriptions (
       id uuid PRIMARY KEY,
       status varchar DEFAULT 'active',
       current_period_start timestamptz,
       current_period_end timestamptz,
       cancel_at_period_end boolean DEFAULT false
    );

   -- Internships table (stores job records saved per user)
    CREATE TABLE IF NOT EXISTS public.internships (
       id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
       user_id uuid REFERENCES public.profiles(id) ON DELETE CASCADE,
       job_title text,
       company_name text,
       application_link text,
       job_description text,
       source_site text DEFAULT 'LinkedIn',
       status varchar DEFAULT 'new',
       notified boolean DEFAULT false,
       created_at timestamptz DEFAULT now(),
       updated_at timestamptz
    );
    ```

6. **Set up Telegram Bot (Optional)**
   
   - Message @BotFather on Telegram
   - Send `/newbot` and follow instructions
   - Get your bot token and chat ID
   - Add them to your secrets.toml or user profile

### Running the Application

```bash
streamlit run app.py
```

The application will be available at `http://localhost:8501`

## Usage

1. **Register/Login**: Create an account or login
2. **Search for Jobs**: Use the scraper to find opportunities
3. **Track Applications**: Manage your application pipeline
4. **Set up Continuous Search**: Get notified about new opportunities
5. **Configure Notifications**: Set up Telegram alerts

## Debugging & runtime artifacts

- When a scraping run encounters an unexpected page layout or LinkedIn blocks the request, the scraper will save debug artifacts to the repository root by default (for example `linkedin_search_results.html` and `linkedin_error.png`).
- To keep the repo clean, the app now writes continuous-search metadata (registry) into a `runtime/` directory. The registry file is:

   - `runtime/continuous_search_registry.json` — a small JSON array used to record active continuous searches started from the Streamlit UI. Each entry includes `user_id`, `job_title`, `location`, `thread_name`, `pid`, and `started_at`.

   Use this file together with the test utility below to inspect live scrapers.

If you prefer debug artifacts to go to `runtime/` as well, I can update `scraper.py` to write debug HTML and screenshots into `runtime/` instead of the repo root.

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SUPABASE_URL` | Your Supabase project URL | Yes |
| `SUPABASE_KEY` | Your Supabase anon key | Yes |

## Security Notes

- Never commit your `.streamlit/secrets.toml` file
- Use environment variables in production
- Keep your Supabase keys secure
- Use service role key for database operations that require elevated permissions


## Support

If you encounter any issues, please create an issue on GitHub with:
- Steps to reproduce
- Error messages
- Your environment details

## Developer utilities

Two small utilities were added to help inspect runtime state and perform a lightweight unused-file scan:

- `test/check_scraper_status.py` — Given a `user_id` it reads `runtime/continuous_search_registry.json` and prints any active continuous-search entries for that user. It will also check the recorded PID if `psutil` is installed.

   Usage:
   ```powershell
   python test/check_scraper_status.py <user_id>
   ```

- `test/find_unused_files.py` — A heuristic static scanner that searches for Python files whose basenames are not referenced anywhere else in the repository. Use it as a starting point to find candidates for cleanup (manual review required).

   Usage:
   ```powershell
   python test/find_unused_files.py
   ```

Notes:
- `psutil` is used by `check_scraper_status.py` to verify whether a recorded process id is alive. It was added to `requirements.txt`. Install dependencies with:

   ```powershell
   pip install -r requirements.txt
   ```

- The scanner is heuristic and will report false positives for files used dynamically or intended as CLI entrypoints — review results before removing files.

If you'd like, I can:
- Move existing LinkedIn HTML fixtures into `test/fixtures/` and add a short README for them, or
- Update `scraper.py` to always write debug artifacts into `runtime/`.

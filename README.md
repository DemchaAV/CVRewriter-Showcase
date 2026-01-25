# CVRewriter 🚀

**CVRewriter** is a full-stack application for **analyzing job vacancy links and generating tailored CVs (PDF)** automatically.  

The project combines **Spring Boot (backend)**, **React (frontend)**, web-scraping, and **AI-assisted content rewriting** to help users adapt their CVs to specific job descriptions.

---

## 📸 Screenshots

### Processing Vacancy
Paste a LinkedIn or Indeed job link → AI analyzes requirements → Generates a tailored PDF resume in seconds.

<p align="center">
  <img src="06_processing.png" alt="Processing vacancy link" width="500"/>
</p>

<p align="center">
  <img src="07_complete.png" alt="Processing complete" width="500"/>
</p>

### History & Records Management
View all your processed vacancies with status tracking (Queued, Completed, Applied, Rejected).

<p align="center">
  <img src="01_history.png" alt="History page" width="800"/>
</p>

Edit vacancy details, load descriptions from links, and manage application status.

<p align="center">
  <img src="02_edit_record.png" alt="Edit record modal" width="400"/>
</p>

### CV Editor
Edit AI-generated CV sections: Technical Skills, Education, Work Experience. Add, remove, or modify items before generating the final PDF.

<p align="center">
  <img src="03_cv_editor.png" alt="CV Editor" width="700"/>
</p>

### Regenerate Feature
Not satisfied with the result? Use the **Regenerate** feature with optional additional instructions to refine your CV.

<p align="center">
  <img src="04_regenerate.png" alt="Regenerate CV" width="700"/>
</p>

### Profile Settings
Manage your CV profile, account settings, and external integrations.

<p align="center">
  <img src="05_profile.png" alt="Profile settings" width="600"/>
</p>

---

## ✨ Key Features

| Feature | Description |
|---------|-------------|
| 🔐 **Authentication** | User registration & login with JWT-based security |
| 🔗 **Vacancy Processing** | Scrape job postings from LinkedIn, Indeed, and other platforms |
| 🤖 **AI-Powered Rewriting** | Multiple AI providers (Gemini, GPT, DeepSeek) analyze vacancy and rewrite CV |
| 📄 **PDF Generation** | Automatic professional CV generation with custom rendering engine |
| ✏️ **CV Editor** | Edit every section of generated CV before final export |
| 🔄 **Regenerate** | Re-run AI with additional instructions for better results |
| 📊 **History Tracking** | View and manage all processed vacancies with status (Applied, Rejected, etc.) |
| 📡 **Real-time Updates** | Server-Sent Events (SSE) for live progress tracking |
| ⚙️ **Token Usage Analytics** | Track AI API token consumption per request |
| 🐳 **Docker Ready** | Full Docker Compose setup for easy deployment |

---

## 🧠 How It Works

```
Step 1: Enter Vacancy Link
     │
     ▼
Step 2: Playwright scrapes job description (LinkedIn/Indeed)
     │
     ▼
Step 3: Gemini AI analyzes requirements & rewrites CV sections
     │
     ▼
Step 4: Edit generated CV in the visual editor
     │
     ▼
Step 5: Download professional PDF resume
```

### Detailed Flow:
1. **User pastes a vacancy URL** (LinkedIn, Indeed, or any job board)
2. **Backend scrapes the job description** using Playwright headless browser
3. **AI (Google Gemini) analyzes** the vacancy requirements and matches them with your profile
4. **CV sections are auto-generated** with relevant keywords and skills
5. **Visual editor** allows manual tweaks before export
6. **PDF is generated** using a custom rendering engine
7. **History saves all records** with status tracking for your job search

---

## 🏗️ Project Structure

```
CVRewriter/
│
├── backend/                    # Spring Boot application
│   ├── src/main/java/com/cvrewriter/
│   │   ├── feature/
│   │   │   ├── ai/             # Gemini AI integration
│   │   │   ├── auth/           # JWT authentication
│   │   │   ├── scraping/       # Playwright web scraping
│   │   │   ├── pdf_render/     # Custom PDF generation
│   │   │   ├── vacancy/        # Vacancy processing logic
│   │   │   ├── resume/         # CV/Resume management
│   │   │   ├── sse/            # Server-Sent Events
│   │   │   ├── token_usage/    # AI token tracking
│   │   │   └── user/           # User management
│   │   └── common/             # Shared utilities
│   ├── prompts/                # AI prompt templates
│   └── pom.xml
│
├── frontend/                   # React application
│   ├── src/
│   │   ├── components/         # Reusable UI components
│   │   ├── pages/              # Application pages
│   │   │   ├── auth/           # Login & Register
│   │   │   ├── processing/     # Vacancy processing
│   │   │   ├── history/        # CV history & editor
│   │   │   └── profile/        # User profile
│   │   ├── services/           # API clients
│   │   └── context/            # React context providers
│   ├── package.json
│   └── vite.config.ts
│
├── screenshots/                # Application screenshots
├── docker-compose.yml          # Full stack deployment
├── .env.docker                 # Docker environment config
└── README.md
```

---

## 🧰 Tech Stack

### Backend
| Technology | Purpose |
|------------|---------|
| Java 17+ | Core language |
| Spring Boot 3.x | Application framework |
| Spring Security | JWT authentication & authorization |
| Spring Data JPA | Database ORM |
| MySQL 8.0 | Primary database |
| Flyway | Database migrations |
| Playwright | Headless browser for web scraping |
| AI Providers | Google Gemini, OpenAI GPT, DeepSeek (configurable) |
| Custom PDF Engine | Advanced PDF document rendering |

### Frontend
| Technology | Purpose |
|------------|---------|
| React 18 | UI framework |
| TypeScript | Type-safe development |
| Vite | Build tool & dev server |
| Axios | HTTP client |
| CSS Modules | Scoped styling |

### DevOps
| Technology | Purpose |
|------------|---------|
| Docker | Containerization |
| Docker Compose | Multi-container orchestration |
| Nginx | Frontend web server & reverse proxy |

---

## ⚙️ Configuration

### Environment Variables

Create `.env.docker` file (or use `.env.docker.example` as template):

```env
# Database
MYSQL_ROOT_PASSWORD=your_password
MYSQL_DATABASE=cv_rewriter

# JWT Security
JWT_SECRET=your_super_secret_jwt_key
JWT_EXPIRATION=86400000
JWT_REFRESH_EXPIRATION=604800000

# Encryption
ENCRYPTION_KEY=16-character-key

# LinkedIn Scraping
LINKEDIN_USERNAME=your_linkedin_email
LINKEDIN_PASSWORD=your_linkedin_password

# AI Providers (configure one or more)
GEMINI_API_KEY=your_gemini_api_key
OPENAI_API_KEY=your_openai_api_key      # Optional
DEEPSEEK_API_KEY=your_deepseek_api_key  # Optional

# Ports
FRONTEND_PORT=3000
BACKEND_PORT=8080
MYSQL_PORT=3307
```

> ⚠️ **Never commit `.env` files to GitHub!**

---

## ▶️ Running the Project

### 🐳 With Docker (Recommended)

```bash
# Clone the repository
git clone https://github.com/yourusername/CVRewriter.git
cd CVRewriter

# Create environment file
cp .env.docker.example .env.docker
# Edit .env.docker with your values

# Start all services
docker-compose up -d

# View logs
docker-compose logs -f
```

**Access the application:**
- 🌐 Frontend: http://localhost:3000
- 🔧 Backend API: http://localhost:8080
- 🗄️ MySQL: localhost:3307

### 🔧 Manual Setup

#### Backend
```bash
cd backend
# Install dependencies
mvn clean install

# Run the application
mvn spring-boot:run
```
👉 Runs on: http://localhost:8080

#### Frontend
```bash
cd frontend
# Install dependencies
npm install

# Run development server
npm run dev
```
👉 Runs on: http://localhost:5173

---

## 🔌 API Endpoints

### Authentication
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | Register new user |
| POST | `/auth/login` | User login |
| POST | `/auth/refresh` | Refresh JWT token |

### Vacancy Processing
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/process` | Process vacancy link (SSE stream) |
| GET | `/records` | Get user's processing history |
| GET | `/records/{id}` | Get specific record details |
| PUT | `/records/{id}` | Update record (status, description) |
| GET | `/records/{id}/download` | Download generated PDF |
| POST | `/records/{id}/regenerate` | Regenerate CV with new instructions |

### User Management
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/user/profile` | Get user profile |
| PUT | `/user/profile` | Update profile |
| GET | `/user/account` | Get account settings |

---

## 🛡️ Security Features

- ✅ Passwords hashed with **BCrypt**
- ✅ JWT-based stateless authentication
- ✅ Sensitive data encrypted at rest
- ✅ LinkedIn sessions isolated via browser contexts
- ✅ CORS configuration for cross-origin requests
- ✅ Input validation and sanitization

---

## 🚧 Development Roadmap

### In Progress
- [ ] Multiple CV template support
- [ ] Cover letter generation
- [ ] Batch processing mode

### Planned
- [ ] OAuth LinkedIn integration
- [ ] Mobile-responsive UI improvements
- [ ] CV templates marketplace
- [ ] Usage analytics dashboard
- [ ] API rate limiting
- [ ] Multi-language support

---

## 📌 Motivation

**CVRewriter** was created to:

- ⏰ **Save time** – Stop manually editing CVs for each application
- 🎯 **Increase relevance** – AI matches your skills to job requirements
- 📈 **Improve ATS scores** – Better keyword optimization
- 🔄 **Automate repetition** – One-click CV generation

---

## 👤 Author

**Artem Demchyshyn**  
Junior Java Backend Developer 🇬🇧🇺🇦

- 💼 Java / Spring Boot / SQL
- 🤖 Automation & AI integrations
- 📧 [Contact via GitHub Issues]

---

## ⭐ Support

If you find this project useful:

- ⭐ **Star** the repository
- 🐛 **Report** bugs via Issues
- 💡 **Suggest** improvements
- 🍴 **Fork** and contribute

---

## 📜 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

<p align="center">
  <strong>CVRewriter</strong> – rewrite smarter, not harder. 🚀
</p>

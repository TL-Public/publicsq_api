
# Question Bank Management System

A production-ready FastAPI application for managing question banks with role-based access control, bulk upload functionality, and organizational context management.

## Features

### Authentication & Authorization
- JWT-based authentication with refresh tokens
- Role-based access control (RBAC)
- Organizational context isolation
- Permission-based endpoint protection

### Bulk Upload System
- Excel file upload with comprehensive validation
- Async job processing with real-time status tracking
- Detailed error reporting and handling
- Template generation for standardized uploads

### Question Management
- Full CRUD operations for questions
- Organizational scope enforcement
- Advanced filtering and search capabilities
- Backward compatibility with legacy systems

### Multi-tenant Architecture
- Organization, Block, and School hierarchy
- User context-based data isolation
- Scalable permission system

## Quick Start

### Prerequisites
- Python 3.11+
- PostgreSQL 12+
- Redis (optional, for caching)

### Installation

1. **Clone and install dependencies:**
   ```bash
   git clone <repository-url>
   cd question-bank-backend
   pip install -r requirements.txt
   ```

2. **Configure environment:**
   ```bash
   cp .env.example .env
   # Edit .env with your database and security settings
   ```

4. **Start the application:**
   ```bash
   uvicorn app.main:app --reload
   ```

#### Authentication
- `POST /v1/login` - User authentication


#### Bulk Upload
- `POST /v1/upload-excel` - Upload Excel file for bulk processing
- `GET /v1/upload-jobs/{job_id}` - Check upload job status
- `GET /v1/excel-template` - Download Excel template

## Development

### Running Tests
```bash
pytest tests/ -v
```

### Code Quality
```bash
# Format code
black app/ tests/

# Lint code
flake8 app/ tests/

# Type checking
mypy app/
```

## Architecture

### Technology Stack
- **Framework**: FastAPI
- **Database**: PostgreSQL with SQLAlchemy ORM
- **Authentication**: JWT with OAuth2
- **Async Processing**: Background tasks for file processing
- **Validation**: Pydantic models
- **Testing**: Pytest with async support

### Project Structure
```
app/
├── api/v1/          # API routes and endpoints
├── models/          # SQLAlchemy database models
├── schemas/         # Pydantic request/response schemas
├── services/        # Business logic layer
├── middleware/      # Custom middleware (RBAC, etc.)
├── utils/           # Utility functions
└── main.py          # Application entry point
```

## Security

- **Authentication**: JWT tokens with configurable expiration
- **Authorization**: Role-based permissions with organizational context
- **Data Protection**: Input validation, SQL injection prevention
- **File Security**: Secure file upload handling
- **Audit Logging**: Comprehensive security event logging

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests for new functionality
5. Ensure all tests pass
6. Submit a pull request

## License

This project is licensed under the MIT License - see the LICENSE file for details.



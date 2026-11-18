# Flask-backend-set-up
## app.py
```
from flask import Flask, request

from config import Config
from extensions import db, migrate, bcrypt, jwt

# Import models to register SQLAlchemy metadata
from models import Patient, Doctor, Appointment

# Import blueprints
from controllers.patient_controller import patient_bp
from controllers.doctor_controller import doctor_bp
from controllers.appointment_controller import appointment_bp

# Import application configuration functions
from errors.handlers import register_error_handlers


app = Flask(__name__)

# 1. Load configuration
app.config.from_object(Config)


# 2. Initialise extensions
db.init_app(app)
migrate.init_app(app, db)
bcrypt.init_app(app)
jwt.init_app(app)


# 3. Register blueprints
app.register_blueprint(patient_bp)
app.register_blueprint(doctor_bp)
app.register_blueprint(appointment_bp)


# 4. Register error handlers
register_error_handlers(app)


# 5. Configure CORS
@app.after_request
def add_cors_headers(response):

    origin = request.headers.get("Origin")

    if origin and origin in app.config["CORS_ORIGINS"]:

        response.headers["Access-Control-Allow-Origin"] = origin
        response.headers["Access-Control-Allow-Headers"] = (
            "Content-Type, Authorization"
        )
        response.headers["Access-Control-Allow-Methods"] = (
            "GET, POST, PATCH, DELETE, OPTIONS"
        )
        response.headers["Access-Control-Max-Age"] = "600"
        response.headers["Vary"] = "Origin"

    return response


# 6. Health check
@app.route("/health", methods=["GET"])
def health_check():

    return {
        "status": "healthy"
    }, 200


if __name__ == "__main__":
    app.run(debug=app.config.get("DEBUG", False))

```
# errors/handlers.py

```
from flask import jsonify

def register_error_handlers(app):

    @app.errorhandler(404)
    def not_found(error):
        return jsonify({
            "error": "Resource not found"
        }), 404

    @app.errorhandler(500)
    def internal_server_error(error):
        return jsonify({
            "error": "Internal server error"
        }), 500

    @app.errorhandler(400)
    def bad_request(error):
        return jsonify({
            "error": "Bad request"
        }), 400

```

# config.py

```
import os


class Config:
    SECRET_KEY = os.environ.get("SECRET_KEY", "dev-only-secret-change-me-in-production-0123456789")
    JWT_SECRET_KEY = os.environ.get("JWT_SECRET_KEY", "dev-only-jwt-secret-change-me-in-production-0123456789")
    JWT_EXPIRES_HOURS = int(os.environ.get("JWT_EXPIRES_HOURS", "8"))

    SQLALCHEMY_DATABASE_URI = os.environ.get("DATABASE_URL", "sqlite:///inventory.db")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    
    CORS_ORIGINS = [
        o.strip()
        for o in os.environ.get(
            "CORS_ORIGINS", "http://localhost:5173,http://127.0.0.1:5173"
        ).split(",")
    ]

```

# extensions.py
```
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_bcrypt import Bcrypt
from flask_jwt_extended import JWTManager

db = SQLAlchemy()
migrate = Migrate()
bcrypt = Bcrypt()
jwt = JWTManager()

```

# requirements.txt

```
flask
flask-sqlalchemy
flask-migrate
marshmallow
flask-marshmallow
marshmallow-sqlalchemy
flask-jwt-extended
flask-bcrypt
flask-cors
pytest
python-dotenv
gunicorn

```

# AUTHS......

## @role_required
```

from functools import wraps
from flask import jsonify, g

def role_required(*required_roles):
    allowed_roles = {role.strip().lower() for role in required_roles}

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):

            # Retrieve authenticated user
            current_user = getattr(g, "current_user", None)

            if current_user is None:
                return jsonify({ "error": "Authentication required" }), 401

            user_role = current_user.role

            if not isinstance(user_role, str):
                return jsonify({ "error": "Invalid user role" }), 403

            user_role = user_role.strip().lower()

            if user_role not in allowed_roles:
                return jsonify({ "error": "You do not have permission to access this resource" }), 403

            return func(*args, **kwargs)
        return wrapper
    return decorator

```

## manage_token
```
from datetime import datetime, timedelta, timezone
import jwt
from flask import current_app


def _secret_key():
	return current_app.config.get("JWT_SECRET_KEY")

def _encode(payload):
	return jwt.encode(payload, _secret_key(), algorithm="HS256")

def _decode(token, expected_type):
	payload = jwt.decode(
		token,
		_secret_key(),
		algorithms=["HS256"],
		options={"require": ["exp", "iat", "sub", "typ"]},
	)
	if payload.get("typ") != expected_type:
		raise jwt.InvalidTokenError("Unexpected token type")

	return payload


def create_token(user):
	now = datetime.now(timezone.utc)
	lifetime_hours = current_app.config.get("JWT_EXPIRES_HOURS", 8)
	payload = {
		"sub": str(user.id),
		"typ": "access",
		"pwd": user.password_hash[-12:],
		"iat": now,
		"exp": now + timedelta(hours=lifetime_hours),
	}
	return _encode(payload)


def decode_token(token):
	return _decode(token, "access")


def create_reset_token(user):
	"""Create a short-lived reset token tied to the current password hash."""
	now = datetime.now(timezone.utc)
	payload = {
		"sub": str(user.id),
		"typ": "password_reset",
		"pwd": user.password_hash[-12:],
		"iat": now,
		"exp": now + timedelta(minutes=10),
	}
	return _encode(payload)


def decode_reset_token(token):
	"""Decode a password-reset token; rejects access tokens and other JWTs."""
	return _decode(token, "password_reset")

```

## @token_required
```

import jwt
from functools import wraps
from flask import request, jsonify, current_app, g
from extensions import db
from models.user import User


def token_required(func):
    @wraps(func)
    def wrapper(*args, **kwargs):

        # Retrieve the Authorization header
        auth_header = request.headers.get("Authorization")

        if not auth_header:
            return jsonify({ "error": "Authorization header missing" }), 401

        parts = auth_header.split()

        if len(parts) != 2 or parts[0].lower() != "bearer":
            return jsonify({ "error": "Invalid authorization header" }), 401

        token = parts[1]

        try:
            decoded_token = jwt.decode( token, current_app.config["JWT_SECRET_KEY"], algorithms=["HS256"])
            user_id = decoded_token.get("sub")

            if not user_id:
                return jsonify({ "error": "Invalid token payload" }), 401

            current_user = db.session.get(User, int(user_id))
            if not current_user:
                return jsonify({ "error": "User no longer exists" }), 401

            #  Store authenticated user for this request
            g.current_user = current_user

        except jwt.ExpiredSignatureError:
            return jsonify({ "error": "Token has expired" }), 401
        except (jwt.InvalidTokenError, ValueError, TypeError):
            return jsonify({ "error": "Invalid authentication token" }), 401

        return func(*args, **kwargs)
    return wrapper

```


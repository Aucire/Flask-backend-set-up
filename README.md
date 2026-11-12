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



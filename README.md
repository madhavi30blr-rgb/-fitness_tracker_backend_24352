# -fitness_tracker_backend_24352
 This is a brief, optional description of the project. A good description would be something like, "A database-driven fitness tracker application with PostgreSQL backend and web frontend."
Flask
psycopg2-binary
import psycopg2
import os

def get_db_connection():
    conn = psycopg2.connect(
        host=os.environ.get("DB_HOST", "localhost"),
        database=os.environ.get("DB_NAME", "fitness_db"),
        user=os.environ.get("DB_USER", "your_username"),
        password=os.environ.get("DB_PASS", "your_password")
    )
    return conn
    from flask import Flask, request, jsonify
from db_config import get_db_connection
import uuid

app = Flask(__name__)

@app.route('/users', methods=['POST'])
def create_user():
    data = request.json
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute(
            "INSERT INTO users (username, email, password_hash) VALUES (%s, %s, %s) RETURNING user_id;",
            (data['username'], data['email'], data['password_hash'])
        )
        user_id = cur.fetchone()[0]
        conn.commit()
        cur.close()
        conn.close()
        return jsonify({"message": "User created successfully!", "user_id": str(user_id)}), 201
    except Exception as e:
        return jsonify({"error": str(e)}), 400

@app.route('/workouts', methods=['POST'])
def log_workout():
    data = request.json
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        
        cur.execute("BEGIN;")

        cur.execute(
            "INSERT INTO workouts (user_id, workout_date, duration_minutes, workout_type) VALUES (%s, %s, %s, %s) RETURNING workout_id;",
            (uuid.UUID(data['user_id']), data['workout_date'], data['duration'], data['type'])
        )
        workout_id = cur.fetchone()[0]

        for ex in data['exercises']:
            cur.execute(
                "INSERT INTO exercises (workout_id, exercise_name, reps, sets, weight_kg) VALUES (%s, %s, %s, %s, %s);",
                (workout_id, ex['name'], ex['reps'], ex['sets'], ex['weight'])
            )

        cur.execute("COMMIT;")
        cur.close()
        conn.close()
        return jsonify({"message": "Workout logged successfully!", "workout_id": str(workout_id)}), 201
    except Exception as e:
        conn.rollback()
        return jsonify({"error": str(e)}), 400

@app.route('/workouts/<user_id>', methods=['GET'])
def get_user_workouts(user_id):
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT * FROM workouts WHERE user_id = %s ORDER BY workout_date DESC;", (uuid.UUID(user_id),))
        workouts = cur.fetchall()
        cur.close()
        conn.close()
        return jsonify(workouts), 200
    except Exception as e:
        return jsonify({"error": str(e)}), 404

if __name__ == '__main__':
    app.run(debug=True)

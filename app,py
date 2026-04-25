from flask import Flask, request, jsonify
import math
import os

app = Flask(__name__)

# ==========================================
# CONFIG
# ==========================================
MAX_SEATS = 8

TIMISOARA = {
    "name": "Timisoara",
    "lat": 45.7489,
    "lng": 21.2087
}

# ==========================================
# HELPERS
# ==========================================
def safe_float(v):
    try:
        if v is None or v == "":
            return None
        return float(v)
    except:
        return None


def safe_int(v, default=1):
    try:
        if v is None or v == "":
            return default
        x = int(v)
        return max(1, x)
    except:
        return default


def haversine(lat1, lon1, lat2, lon2):
    R = 6371

    dlat = math.radians(lat2 - lat1)
    dlon = math.radians(lon2 - lon1)

    a = (
        math.sin(dlat / 2) ** 2
        + math.cos(math.radians(lat1))
        * math.cos(math.radians(lat2))
        * math.sin(dlon / 2) ** 2
    )

    c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))
    return R * c


# ==========================================
# NORMALIZE INPUT
# ==========================================
def normalize_input(data):
    if isinstance(data, dict) and "bookings" in data:
        return data["bookings"]

    if isinstance(data, list):
        return data

    return []


# ==========================================
# PREPARE BOOKINGS
# ==========================================
def prepare(bookings):
    cleaned = []
    skipped = []

    for b in bookings:
        drop_lat = safe_float(b.get("drop_lat"))
        drop_lng = safe_float(b.get("drop_lng"))

        if drop_lat is None or drop_lng is None:
            skipped.append({
                "id": b.get("id"),
                "reason": "missing drop coordinates"
            })
            continue

        persons = safe_int(b.get("persons"), 1)

        if persons > MAX_SEATS:
            persons = MAX_SEATS

        cleaned.append({
            "id": b.get("id"),
            "name": b.get("name", ""),
            "pickup_address": b.get("pickup_address", "Timisoara"),
            "pickup_lat": TIMISOARA["lat"],
            "pickup_lng": TIMISOARA["lng"],
            "dropoff_address": b.get("dropoff_address", ""),
            "drop_lat": drop_lat,
            "drop_lng": drop_lng,
            "persons": persons,
            "phone": b.get("phone", ""),
            "price": b.get("price", ""),
            "notes": b.get("notes", "")
        })

    return cleaned, skipped


# ==========================================
# GROUPING BY DESTINATION
# ==========================================
def optimize_routes(bookings):
    """
    Grupează după direcție + apropiere destinație
    plecare comună: Timișoara
    """

    for b in bookings:
        b["distance_from_tm"] = haversine(
            TIMISOARA["lat"],
            TIMISOARA["lng"],
            b["drop_lat"],
            b["drop_lng"]
        )

    # sortăm departe -> aproape
    bookings = sorted(
        bookings,
        key=lambda x: x["distance_from_tm"],
        reverse=True
    )

    cars = []

    for booking in bookings:
        placed = False

        for car in cars:
            used = sum(x["persons"] for x in car)

            if used + booking["persons"] > MAX_SEATS:
                continue

            # comparăm cu ultimul drop din mașină
            last = car[-1]

            km_between = haversine(
                last["drop_lat"],
                last["drop_lng"],
                booking["drop_lat"],
                booking["drop_lng"]
            )

            # dacă destinația e apropiată
            if km_between <= 180:
                car.append(booking)
                placed = True
                break

        if not placed:
            cars.append([booking])

    return cars


# ==========================================
# EXPORT
# ==========================================
def export_person(x):
    return {
        "id": x["id"],
        "name": x["name"],
        "persons": x["persons"],
        "pickup_address": x["pickup_address"],
        "pickup_lat": x["pickup_lat"],
        "pickup_lng": x["pickup_lng"],
        "dropoff_address": x["dropoff_address"],
        "drop_lat": x["drop_lat"],
        "drop_lng": x["drop_lng"],
        "phone": x["phone"],
        "price": x["price"],
        "notes": x["notes"]
    }


# ==========================================
# ROUTES
# ==========================================
@app.route("/", methods=["GET"])
def home():
    return jsonify({
        "status": "online",
        "message": "Dropoff Optimizer from Timisoara"
    })


@app.route("/optimize", methods=["GET", "POST"])
def optimize():
    try:
        if request.method == "GET":
            return jsonify({
                "status": "online",
                "message": "Use POST with bookings JSON"
            })

        raw = request.get_json(silent=True)
        bookings = normalize_input(raw)

        if not bookings:
            return jsonify({
                "status": "error",
                "message": "No bookings found"
            }), 400

        prepared, skipped = prepare(bookings)

        if not prepared:
            return jsonify({
                "status": "error",
                "message": "No valid bookings"
            }), 400

        routes = optimize_routes(prepared)

        cars = []

        for i, route in enumerate(routes, start=1):
            cars.append({
                "car_number": i,
                "seats_used": sum(x["persons"] for x in route),
                "total_stops": len(route),
                "route": [export_person(x) for x in route]
            })

        return jsonify({
            "status": "success",
            "start_point": "Timisoara",
            "total_received": len(bookings),
            "valid_bookings": len(prepared),
            "skipped_bookings": len(skipped),
            "skipped_details": skipped,
            "total_cars": len(cars),
            "cars": cars
        })

    except Exception as e:
        return jsonify({
            "status": "error",
            "message": str(e)
        }), 500


# ==========================================
# RUN
# ==========================================
if __name__ == "__main__":
    app.run(
        host="0.0.0.0",
        port=int(os.environ.get("PORT", 5000))
    )

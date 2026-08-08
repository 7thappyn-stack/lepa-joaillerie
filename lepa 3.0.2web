import os
import random
import sqlite3
from datetime import datetime

from flask import (
    Flask,
    flash,
    jsonify,
    redirect,
    render_template_string,
    request,
    session,
    url_for,
)
from werkzeug.utils import secure_filename

# --- CONFIGURATION PRODUCTION & BASE DE DONNÉES ---
BASE_DIR = os.path.abspath(os.path.dirname(__file__))
UPLOAD_FOLDER = os.path.join(BASE_DIR, "static")
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

app = Flask(__name__, static_folder=UPLOAD_FOLDER)
app.secret_key = os.environ.get("SECRET_KEY", "lepa_secret_key_ultra_secure_2026")
app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER

DB_NAME = os.path.join(BASE_DIR, "lepa.db")


# Filtre personnalisé pour afficher les prix avec séparateur de milliers
@app.template_filter("format_price")
def format_price(value):
  try:
    if value is None:
      return "0"
    val_int = int(round(float(value)))
    return f"{val_int:,}".replace(",", ".")
  except (ValueError, TypeError):
    return str(value)


def get_db():
  conn = sqlite3.connect(DB_NAME)
  conn.row_factory = sqlite3.Row
  return conn


def init_db():
  with get_db() as db:
    db.execute("""
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                email TEXT UNIQUE NOT NULL,
                phone TEXT NOT NULL,
                gender TEXT NOT NULL DEFAULT 'homme',
                is_admin BOOLEAN DEFAULT 0
            )
        """)

    db.execute("""
            CREATE TABLE IF NOT EXISTS settings (
                key TEXT PRIMARY KEY,
                value TEXT NOT NULL
            )
        """)
    
    db.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('admin_code', 'LEPA2026')")
    db.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('support_phone', '+225 00 00 00 00')")
    db.execute("INSERT OR IGNORE INTO settings (key, value) VALUES ('support_email', 'support@lepa-jewelry.com')")

    db.execute("""
            CREATE TABLE IF NOT EXISTS moderators (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                code TEXT UNIQUE NOT NULL,
                is_active BOOLEAN DEFAULT 1
            )
        """)

    db.execute("""
            CREATE TABLE IF NOT EXISTS banners (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                image TEXT NOT NULL,
                title TEXT DEFAULT ''
            )
        """)

    try:
      db.execute(
          "ALTER TABLE users ADD COLUMN gender TEXT NOT NULL DEFAULT 'homme'"
      )
    except sqlite3.OperationalError:
      pass

    db.execute("""
            CREATE TABLE IF NOT EXISTS products (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                category TEXT NOT NULL,
                price REAL NOT NULL,
                image TEXT DEFAULT 'default.jpg',
                in_stock BOOLEAN DEFAULT 1
            )
        """)
    db.execute("""
            CREATE TABLE IF NOT EXISTS orders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                product_id INTEGER NOT NULL,
                address TEXT,
                distance REAL NOT NULL,
                shipping_fee REAL NOT NULL,
                total_price REAL NOT NULL,
                status TEXT DEFAULT 'En attente de validation admin'
            )
        """)
    db.execute("""
            CREATE TABLE IF NOT EXISTS reservations (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                product_id INTEGER NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                note TEXT
            )
        """)
    db.execute("""
            CREATE TABLE IF NOT EXISTS reviews (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                product_id INTEGER NOT NULL,
                user_id INTEGER NOT NULL,
                rating INTEGER NOT NULL,
                comment TEXT NOT NULL
            )
        """)
    db.execute("""
            CREATE TABLE IF NOT EXISTS suggestions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                message TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
    db.execute("""
            CREATE TABLE IF NOT EXISTS claims (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                message TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
    db.execute("""
            CREATE TABLE IF NOT EXISTS bugs (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                description TEXT NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        """)
    db.execute("""
            CREATE TABLE IF NOT EXISTS inbox (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                title TEXT NOT NULL,
                content TEXT NOT NULL,
                is_read BOOLEAN DEFAULT 0
            )
        """)
    db.commit()


@app.context_processor
def inject_globals():
  unread_count = 0
  if "user_id" in session:
    try:
      with get_db() as db:
        res = db.execute(
            "SELECT COUNT(*) FROM inbox WHERE user_id = ? AND is_read = 0",
            (session["user_id"],),
        ).fetchone()
        if res:
          unread_count = res[0]
    except sqlite3.OperationalError:
      unread_count = 0
      
  support_info = {"phone": "+225 00 00 00 00", "email": "support@lepa-jewelry.com"}
  try:
    with get_db() as db:
      p = db.execute("SELECT value FROM settings WHERE key = 'support_phone'").fetchone()
      e = db.execute("SELECT value FROM settings WHERE key = 'support_email'").fetchone()
      if p: support_info["phone"] = p["value"]
      if e: support_info["email"] = e["value"]
  except Exception:
    pass

  return dict(unread_inbox_count=unread_count, support_info=support_info)


# ==================== ROUTES DE LA BOUTIQUE ====================

@app.route("/")
def index():
  category = request.args.get("category")
  with get_db() as db:
    if category:
      products = db.execute(
          "SELECT * FROM products WHERE category = ?", (category,)
      ).fetchall()
    else:
      products = db.execute("SELECT * FROM products").fetchall()

    banners = db.execute("SELECT * FROM banners").fetchall()
    banners_list = [dict(b) for b in banners]
    random.shuffle(banners_list)

  favorites = session.get("favorites", [])
  return render_template_string(
      HOME_TEMPLATE,
      products=products,
      current_category=category,
      favorites=favorites,
      banners=banners_list,
  )


@app.route("/api/live_data")
def live_data():
  category = request.args.get("category")
  with get_db() as db:
    if category:
      products = db.execute(
          "SELECT * FROM products WHERE category = ?", (category,)
      ).fetchall()
    else:
      products = db.execute("SELECT * FROM products").fetchall()
    
    banners = db.execute("SELECT * FROM banners").fetchall()
    
  favorites = session.get("favorites", [])
  
  prod_list = []
  for p in products:
    img_src = p["image"] if p["image"].startswith('http') else (url_for('static', filename=p["image"]) if p["image"] != 'default.jpg' else 'https://images.unsplash.com/photo-1515562141207-7a88fb7ce338?w=300')
    val_int = int(round(float(p["price"])))
    formatted_price = f"{val_int:,}".replace(",", ".")
    
    prod_list.append({
      "id": p["id"],
      "name": p["name"],
      "category": p["category"],
      "price": formatted_price,
      "image": img_src,
      "in_stock": bool(p["in_stock"]),
      "is_fav": p["id"] in favorites
    })
    
  banner_list = []
  for b in banners:
    img_b = b["image"] if b["image"].startswith('http') else url_for('static', filename=b["image"])
    banner_list.append({
      "id": b["id"],
      "image": img_b,
      "title": b["title"] or ""
    })
    
  return jsonify({"products": prod_list, "banners": banner_list})


@app.route("/toggle_favorite/<int:product_id>", methods=["POST"])
def toggle_favorite(product_id):
  if "favorites" not in session:
    session["favorites"] = []
  favs = list(session["favorites"])

  if product_id in favs:
    favs.remove(product_id)
    added = False
  else:
    favs.append(product_id)
    added = True

  session["favorites"] = favs
  session.modified = True
  return jsonify(
      {"success": True, "added": added, "count": len(session["favorites"])}
  )


@app.route("/favorites")
def favorites_page():
  fav_ids = session.get("favorites", [])
  products = []
  if fav_ids:
    with get_db() as db:
      placeholders = ",".join(["?"] * len(fav_ids))
      products = db.execute(
          f"SELECT * FROM products WHERE id IN ({placeholders})", fav_ids
      ).fetchall()
  return render_template_string(
      FAVORITES_TEMPLATE, products=products, favorites=fav_ids
  )


@app.route("/register", methods=["GET", "POST"])
def register():
  if request.method == "POST":
    name = request.form["name"].strip()
    email = request.form["email"].strip().lower()
    phone = request.form["phone"].strip()
    gender = request.form.get("gender", "homme")

    with get_db() as db:
      existing = db.execute(
          "SELECT * FROM users WHERE email = ?", (email,)
      ).fetchone()
      if existing:
        flash("Cet email est déjà enregistré !", "danger")
        return redirect(url_for("register"))

      user_count = db.execute("SELECT COUNT(*) FROM users").fetchone()[0]
      is_admin = 1 if user_count == 0 else 0

      cursor = db.cursor()
      cursor.execute(
          "INSERT INTO users (name, email, phone, gender, is_admin) VALUES"
          " (?, ?, ?, ?, ?)",
          (name, email, phone, gender, is_admin),
      )
      db.commit()
      user_id = cursor.lastrowid

    session["user_id"] = user_id
    session["user_name"] = name
    session["user_gender"] = gender
    session["is_admin"] = bool(is_admin)

    salutation = "Monsieur" if gender == "homme" else "Madame"
    flash(f"Inscription réussie, bienvenue {salutation} {name} !", "success")
    return redirect(url_for("index"))
  return render_template_string(REGISTER_TEMPLATE)


@app.route("/login", methods=["GET", "POST"])
def login():
  if request.method == "POST":
    email = request.form["email"].strip().lower()
    phone = request.form["phone"].strip()
    with get_db() as db:
      user = db.execute(
          "SELECT * FROM users WHERE email = ? AND phone = ?", (email, phone)
      ).fetchone()

    if user:
      session["user_id"] = user["id"]
      session["user_name"] = user["name"]
      session["user_gender"] = (
          user["gender"]
          if ("gender" in user.keys() and user["gender"])
          else "homme"
      )
      session["is_admin"] = bool(user["is_admin"])

      salutation = "Monsieur" if session["user_gender"] == "homme" else "Madame"
      flash(f"Ravi de vous revoir, {salutation} {user['name']} !", "success")
      return redirect(url_for("index"))
    flash("Identifiants incorrects (Vérifiez email et téléphone).", "danger")
  return render_template_string(LOGIN_TEMPLATE)


@app.route("/logout")
def logout():
  session.clear()
  flash("Vous êtes déconnecté.", "success")
  return redirect(url_for("index"))


# ==================== PANIER, RESERVATION & COMMANDES ====================

@app.route("/add_to_cart/<int:product_id>")
def add_to_cart(product_id):
  with get_db() as db:
    product = db.execute(
        "SELECT * FROM products WHERE id = ?", (product_id,)
    ).fetchone()

  if not product or not product["in_stock"]:
    flash("Désolé, cet article est en rupture de stock.", "danger")
    return redirect(url_for("product_detail", product_id=product_id))

  if "cart" not in session:
    session["cart"] = []
  session["cart"].append(product_id)
  session.modified = True
  flash(f"'{product['name']}' a été ajouté à votre panier !", "cart_success")

  referrer = request.referrer
  if referrer and "product" in referrer:
    return redirect(referrer)

  return redirect(url_for("product_detail", product_id=product_id))


@app.route("/remove_from_cart/<int:product_id>")
def remove_from_cart(product_id):
  if "cart" in session:
    cart = list(session["cart"])
    if product_id in cart:
      cart.remove(product_id)
      session["cart"] = cart
      session.modified = True
      flash("Article retiré du panier.", "info")
  return redirect(url_for("cart"))


@app.route("/reserve/<int:product_id>", methods=["POST"])
def reserve_product(product_id):
  if "user_id" not in session:
    flash("Veuillez vous connecter pour réserver cet article.", "danger")
    return redirect(url_for("login"))

  with get_db() as db:
    res_count = db.execute(
        "SELECT COUNT(*) FROM reservations WHERE product_id = ?", (product_id,)
    ).fetchone()[0]
    if res_count >= 100:
      flash(
          "La limite maximale de 100 réservations pour cet article a été"
          " atteinte.",
          "danger",
      )
      return redirect(url_for("product_detail", product_id=product_id))

    already = db.execute(
        "SELECT * FROM reservations WHERE user_id = ? AND product_id = ?",
        (session["user_id"], product_id),
    ).fetchone()
    if already:
      flash("Vous avez déjà une réservation en cours pour ce bijou.", "info")
      return redirect(url_for("product_detail", product_id=product_id))

    note = request.form.get("note", "")
    db.execute(
        "INSERT INTO reservations (user_id, product_id, note) VALUES (?, ?,"
        " ?)",
        (session["user_id"], product_id, note),
    )
    db.commit()

  flash(
      "Votre réservation a été transmise avec succès à l'administration !",
      "success",
  )
  return redirect(url_for("product_detail", product_id=product_id))


@app.route("/cart")
def cart():
  if "cart" not in session:
    session["cart"] = []
  product_ids = session["cart"]
  products = []
  total_price = 0
  if product_ids:
    with get_db() as db:
      for pid in product_ids:
        p = db.execute("SELECT * FROM products WHERE id = ?", (pid,)).fetchone()
        if p:
          products.append(p)
      total_price = sum(p["price"] for p in products)
  return render_template_string(
      CART_TEMPLATE, products=products, total_price=total_price
  )


@app.route("/clear_cart")
def clear_cart():
  session.pop("cart", None)
  return redirect(url_for("cart"))


@app.route("/checkout", methods=["GET", "POST"])
def checkout():
  if "user_id" not in session:
    flash("Veuillez vous connecter pour valider votre commande.", "danger")
    return redirect(url_for("login"))

  if "cart" not in session or not session["cart"]:
    flash("Votre panier est vide.", "danger")
    return redirect(url_for("index"))

  product_ids = session["cart"]

  with get_db() as db:
    products = []
    for pid in product_ids:
      p = db.execute("SELECT * FROM products WHERE id = ?", (pid,)).fetchone()
      if p:
        products.append(p)

    if request.method == "POST":
      address = request.form.get("address", "").strip()
      distance = float(request.form["distance"])

      subtotal = sum(p["price"] for p in products)
      shipping_fee = 2000 + (distance * 300) + (subtotal * 0.01)

      for p in products:
        db.execute(
            "INSERT INTO orders (user_id, product_id, address, distance,"
            " shipping_fee, total_price) VALUES (?, ?, ?, ?, ?, ?)",
            (
                session["user_id"],
                p["id"],
                address,
                distance,
                shipping_fee / len(products),
                p["price"] + (shipping_fee / len(products)),
            ),
        )

      db.commit()
      session.pop("cart", None)
      formatted_shipping = f"{int(round(shipping_fee)):,}".replace(",", ".")
      flash(
          f"Commande validée ! Frais de livraison estimés : {formatted_shipping}"
          " FCFA.",
          "success",
      )
      return redirect(url_for("my_orders"))

    subtotal = sum(p["price"] for p in products)

  return render_template_string(
      CHECKOUT_TEMPLATE, products=products, subtotal=subtotal
  )


@app.route("/my_orders")
def my_orders():
  if "user_id" not in session:
    return redirect(url_for("login"))
  with get_db() as db:
    orders = db.execute(
        "SELECT o.*, p.name as product_name, p.image as product_image, p.price"
        " as product_price FROM orders o JOIN products p ON o.product_id = p.id"
        " WHERE o.user_id = ? ORDER BY o.id DESC",
        (session["user_id"],),
    ).fetchall()
  return render_template_string(ORDERS_TEMPLATE, orders=orders)


@app.route("/cancel_order/<int:order_id>", methods=["POST"])
def cancel_order(order_id):
  if "user_id" not in session:
    return redirect(url_for("login"))
  with get_db() as db:
    order = db.execute(
        "SELECT * FROM orders WHERE id = ? AND user_id = ?",
        (order_id, session["user_id"]),
    ).fetchone()
    if order:
      db.execute(
          "UPDATE orders SET status = 'Annulée par le client' WHERE id = ?",
          (order_id,),
      )
      db.commit()
      flash("Votre commande a bien été annulée.", "info")
  return redirect(url_for("my_orders"))


# ==================== CONFIRMATION DE COMMANDE (ADMIN / MODÉRATEUR) ====================

@app.route("/admin/confirm_order/<int:order_id>", methods=["POST"])
def confirm_order(order_id):
  if not session.get("is_admin") and not session.get("is_moderator"):
    flash("Accès non autorisé.", "danger")
    return redirect(url_for("admin_panel"))

  author = session.get("mod_name") if session.get("is_moderator") else "Administrateur"
  new_status = f"Validée par {author}"

  with get_db() as db:
    db.execute("UPDATE orders SET status = ? WHERE id = ?", (new_status, order_id))
    db.commit()
    flash(f"Commande #{order_id} confirmée avec succès.", "success")

  return redirect(url_for("admin_panel"))


# ==================== CLIENT : AVIS, MESSAGES, SAV & BUGS ====================

@app.route("/product/<int:product_id>", methods=["GET", "POST"])
def product_detail(product_id):
  with get_db() as db:
    product = db.execute(
        "SELECT * FROM products WHERE id = ?", (product_id,)
    ).fetchone()
    reviews = db.execute(
        "SELECT * FROM reviews WHERE product_id = ?", (product_id,)
    ).fetchall()
    res_count = db.execute(
        "SELECT COUNT(*) FROM reservations WHERE product_id = ?", (product_id,)
    ).fetchone()[0]

    if request.method == "POST" and "user_id" in session:
      has_bought = db.execute(
          "SELECT * FROM orders WHERE user_id = ? AND product_id = ?",
          (session["user_id"], product_id),
      ).fetchone()
      if has_bought:
        db.execute(
            "INSERT INTO reviews (product_id, user_id, rating, comment) VALUES"
            " (?, ?, ?, ?)",
            (
                product_id,
                session["user_id"],
                int(request.form["rating"]),
                request.form["comment"],
            ),
        )
        db.commit()
        flash("Votre avis a bien été publié !", "success")
      else:
        flash(
            "Vous devez avoir acheté ce bijou pour pouvoir laisser un avis.",
            "danger",
        )
      return redirect(url_for("product_detail", product_id=product_id))

  favorites = session.get("favorites", [])
  return render_template_string(
      PRODUCT_DETAIL_TEMPLATE,
      product=product,
      reviews=reviews,
      favorites=favorites,
      res_count=res_count,
  )


@app.route("/inbox")
def inbox():
  if "user_id" not in session:
    return redirect(url_for("login"))
  with get_db() as db:
    messages = db.execute(
        "SELECT * FROM inbox WHERE user_id = ? ORDER BY id DESC",
        (session["user_id"],),
    ).fetchall()
    db.execute(
        "UPDATE inbox SET is_read = 1 WHERE user_id = ?", (session["user_id"],)
    )
    db.commit()
  return render_template_string(INBOX_TEMPLATE, messages=messages)


@app.route("/suggestions", methods=["GET", "POST"])
def suggestions():
  if "user_id" not in session:
    return redirect(url_for("login"))
  if request.method == "POST":
    with get_db() as db:
      db.execute(
          "INSERT INTO suggestions (user_id, message) VALUES (?, ?)",
          (session["user_id"], request.form["message"]),
      )
      db.commit()
    flash("Merci ! Votre suggestion a bien été transmise.", "success")
  return render_template_string(SUGGESTIONS_TEMPLATE)


@app.route("/claims", methods=["GET", "POST"])
def claims():
  if "user_id" not in session:
    return redirect(url_for("login"))
  if request.method == "POST":
    with get_db() as db:
      db.execute(
          "INSERT INTO claims (user_id, message) VALUES (?, ?)",
          (session["user_id"], request.form["message"]),
      )
      db.commit()
    flash("Votre réclamation a bien été reçue par le service client.", "success")
  return render_template_string(CLAIMS_TEMPLATE)


@app.route("/report_bug", methods=["GET", "POST"])
def report_bug():
  if request.method == "POST":
    description = request.form.get("description", "").strip()
    user_id = session.get("user_id")
    if description:
      with get_db() as db:
        db.execute(
            "INSERT INTO bugs (user_id, description) VALUES (?, ?)",
            (user_id, description),
        )
        db.commit()
      flash(
          "Merci ! Le problème a été signalé à l'équipe technique.", "success"
      )
      return redirect(url_for("index"))
    flash("Veuillez décrire le problème rencontré.", "danger")
  return render_template_string(REPORT_BUG_TEMPLATE)


@app.route("/support")
def customer_service():
  return render_template_string(CUSTOMER_SERVICE_TEMPLATE)


# ==================== PANNEAU ADMINISTRATEUR & MODÉRATEURS ====================

@app.route("/admin_logout", methods=["POST"])
def admin_logout():
  session.pop("is_admin", None)
  session.pop("is_moderator", None)
  session.pop("mod_name", None)
  flash("Vous êtes déconnecté de l'espace administration.", "success")
  return redirect(url_for("admin_panel"))


@app.route("/admin", methods=["GET", "POST"])
def admin_panel():
  admin_code = request.form.get("code")
  
  if admin_code:
    with get_db() as db:
      main_code_db = db.execute("SELECT value FROM settings WHERE key = 'admin_code'").fetchone()
      real_admin_pass = main_code_db["value"] if main_code_db else "LEPA2026"
      
      if admin_code == real_admin_pass:
        session["is_admin"] = True
        session["is_moderator"] = False
        flash("Connexion administrateur réussie.", "success")
        return redirect(url_for("admin_panel"))
      else:
        mod = db.execute("SELECT * FROM moderators WHERE code = ? AND is_active = 1", (admin_code,)).fetchone()
        if mod:
          session["is_admin"] = False
          session["is_moderator"] = True
          session["mod_name"] = mod["name"]
          flash(f"Connexion modérateur réussie ({mod['name']}).", "success")
          return redirect(url_for("admin_panel"))
        else:
          flash("Code d'accès incorrect ou compte désactivé/banni.", "danger")

  if session.get("is_admin") or session.get("is_moderator"):
    is_main_admin = session.get("is_admin", False)
    with get_db() as db:
      if request.method == "POST":
        if "add_banner" in request.form:
          banner_path = ""
          file = request.files.get("banner_file")
          if file and file.filename != "":
            filename = secure_filename(file.filename)
            filename = f"banner_{int(datetime.now().timestamp())}_{filename}"
            file.save(os.path.join(app.config["UPLOAD_FOLDER"], filename))
            banner_path = filename
          elif request.form.get("banner_url"):
            banner_path = request.form.get("banner_url")

          if banner_path:
            title = request.form.get("banner_title", "")
            db.execute("INSERT INTO banners (image, title) VALUES (?, ?)", (banner_path, title))
            db.commit()
            flash("Nouvelle photo de bannière ajoutée !", "success")
            return redirect(url_for("admin_panel"))

        elif "delete_banner" in request.form:
          banner_id = int(request.form["banner_id"])
          db.execute("DELETE FROM banners WHERE id = ?", (banner_id,))
          db.commit()
          flash("Bannière supprimée avec succès.", "success")
          return redirect(url_for("admin_panel"))

        elif "reset_counters" in request.form and is_main_admin:
          db.execute("DELETE FROM orders")
          db.execute("DELETE FROM reservations")
          db.execute("DELETE FROM claims")
          db.execute("DELETE FROM suggestions")
          db.execute("DELETE FROM bugs")
          db.commit()
          flash("Les statistiques ont été remises à zéro !", "success")
          return redirect(url_for("admin_panel"))

        if is_main_admin:
          if "update_settings" in request.form:
            new_phone = request.form.get("support_phone")
            new_email = request.form.get("support_email")
            new_pass = request.form.get("admin_password")
            
            if new_phone:
              db.execute("UPDATE settings SET value = ? WHERE key = 'support_phone'", (new_phone,))
            if new_email:
              db.execute("UPDATE settings SET value = ? WHERE key = 'support_email'", (new_email,))
            if new_pass:
              db.execute("UPDATE settings SET value = ? WHERE key = 'admin_code'", (new_pass,))
            db.commit()
            flash("Paramètres mis à jour avec succès !", "success")
            return redirect(url_for("admin_panel"))

          elif "add_moderator" in request.form:
            mod_name = request.form.get("mod_name")
            mod_code = request.form.get("mod_code")
            if mod_name and mod_code:
              try:
                db.execute("INSERT INTO moderators (name, code, is_active) VALUES (?, ?, 1)", (mod_name, mod_code))
                db.commit()
                flash(f"Modérateur '{mod_name}' ajouté avec succès !", "success")
              except sqlite3.IntegrityError:
                flash("Ce code modérateur existe déjà.", "danger")
            return redirect(url_for("admin_panel"))

          elif "toggle_moderator" in request.form:
            mod_id = int(request.form["mod_id"])
            current_mod = db.execute("SELECT is_active FROM moderators WHERE id = ?", (mod_id,)).fetchone()
            new_status = 0 if current_mod and current_mod["is_active"] else 1
            db.execute("UPDATE moderators SET is_active = ? WHERE id = ?", (new_status, mod_id))
            db.commit()
            flash("Statut du modérateur mis à jour.", "info")
            return redirect(url_for("admin_panel"))

          elif "delete_moderator" in request.form:
            mod_id = int(request.form["mod_id"])
            db.execute("DELETE FROM moderators WHERE id = ?", (mod_id,))
            db.commit()
            flash("Compte modérateur supprimé.", "success")
            return redirect(url_for("admin_panel"))

        if "add_product" in request.form:
          image_path = "default.jpg"
          file = request.files.get("image_file")
          if file and file.filename != "":
            filename = secure_filename(file.filename)
            filename = f"{int(datetime.now().timestamp())}_{filename}"
            file.save(os.path.join(app.config["UPLOAD_FOLDER"], filename))
            image_path = filename
          elif request.form.get("image_url"):
            image_path = request.form.get("image_url")

          in_stock = 1 if "in_stock" in request.form else 0
          cursor = db.cursor()
          cursor.execute(
              "INSERT INTO products (name, category, price, image, in_stock)"
              " VALUES (?, ?, ?, ?, ?)",
              (
                  request.form["name"],
                  request.form["category"],
                  float(request.form["price"]),
                  image_path,
                  in_stock,
              ),
          )
          db.commit()
          new_prod_id = cursor.lastrowid
          
          if request.headers.get("X-Requested-With") == "XMLHttpRequest" or (request.accept_mimetypes.accept_json and not request.accept_mimetypes.accept_html):
            img_src = image_path if image_path.startswith('http') else (url_for('static', filename=image_path) if image_path != 'default.jpg' else 'https://images.unsplash.com/photo-1515562141207-7a88fb7ce338?w=100')
            formatted_price = f"{int(round(float(request.form['price']))):,}".replace(",", ".") + " FCFA"
            stock_text = "En stock" if in_stock else "Rupture"
            stock_color = "#a3e635" if in_stock else "#ef4444"
            
            html_row = f"""
            <tr id="product-row-{new_prod_id}">
              <td><img src="{img_src}" style="width:35px; height:35px; object-fit:cover; border-radius:3px;"></td>
              <td>{request.form['name']}</td>
              <td>{request.form['category']}</td>
              <td>{formatted_price}</td>
              <td><span style="color: {stock_color}; font-weight:bold;">{stock_text}</span></td>
              <td>
                <form method="POST" style="display:inline;">
                  <input type="hidden" name="product_id" value="{new_prod_id}">
                  <button type="submit" name="toggle_stock" class="btn" style="padding:4px 8px; font-size:11px; background:#444; color:#fff;">Inverser Stock</button>
                  <button type="submit" name="delete_product" class="btn" style="padding:4px 8px; font-size:11px; background:#991b1b; color:#fff;" onclick="return confirm('Supprimer ?');">Supprimer</button>
                </form>
              </td>
            </tr>
            """
            return jsonify({"success": True, "html": html_row, "message": "Ajouté avec succès !"})

          flash("Nouveau bijou ajouté avec succès !", "success")
          return redirect(url_for("admin_panel"))

        elif "toggle_stock" in request.form:
          prod_id = int(request.form["product_id"])
          current = db.execute(
              "SELECT in_stock FROM products WHERE id = ?", (prod_id,)
          ).fetchone()
          new_status = 0 if current and current["in_stock"] else 1
          db.execute(
              "UPDATE products SET in_stock = ? WHERE id = ?",
              (new_status, prod_id),
          )
          db.commit()
          flash("Statut du stock mis à jour !", "success")
          return redirect(url_for("admin_panel"))

        elif "delete_product" in request.form:
          prod_id = int(request.form["product_id"])
          db.execute("DELETE FROM products WHERE id = ?", (prod_id,))
          db.commit()
          flash("Article supprimé avec succès.", "success")
          return redirect(url_for("admin_panel"))

        elif "send_inbox" in request.form:
          db.execute(
              "INSERT INTO inbox (user_id, title, content) VALUES (?, ?, ?)",
              (
                  int(request.form["user_id"]),
                  request.form["title"],
                  request.form["content"],
              ),
          )
          db.commit()
          flash("Message envoyé dans la boîte du client.", "success")
          return redirect(url_for("admin_panel"))

      users_count = db.execute("SELECT COUNT(*) FROM users").fetchone()[0]
      users = db.execute("SELECT * FROM users").fetchall()
      products = db.execute("SELECT * FROM products").fetchall()
      moderators = db.execute("SELECT * FROM moderators").fetchall()
      banners = db.execute("SELECT * FROM banners").fetchall()

      current_settings = {}
      settings_rows = db.execute("SELECT * FROM settings").fetchall()
      for row in settings_rows:
        current_settings[row["key"]] = row["value"]

      orders = db.execute(
          "SELECT o.*, u.name as user_name, u.phone as user_phone, p.name as"
          " product_name, p.image as product_image FROM orders o JOIN users u ON"
          " o.user_id = u.id JOIN products p ON o.product_id = p.id ORDER BY"
          " o.id DESC"
      ).fetchall()

      reservations = db.execute(
          "SELECT r.*, u.name as user_name, u.phone as user_phone, p.name as"
          " product_name, p.image as product_image FROM reservations r JOIN"
          " users u ON r.user_id = u.id JOIN products p ON r.product_id = p.id"
          " ORDER BY r.id DESC"
      ).fetchall()

      suggestions = db.execute(
          "SELECT s.*, u.name as user_name FROM suggestions s LEFT JOIN users u ON s.user_id = u.id ORDER BY s.id DESC"
      ).fetchall()
      
      claims = db.execute(
          "SELECT c.*, u.name as user_name, u.phone as user_phone FROM claims c LEFT JOIN users u ON c.user_id = u.id ORDER BY c.id DESC"
      ).fetchall()
      
      reported_bugs = db.execute(
          "SELECT b.*, u.name as user_name FROM bugs b LEFT JOIN users u ON"
          " b.user_id = u.id ORDER BY b.id DESC"
      ).fetchall()

    return render_template_string(
        ADMIN_TEMPLATE,
        users_count=users_count,
        users=users,
        products=products,
        orders=orders,
        reservations=reservations,
        suggestions=suggestions,
        claims=claims,
        reported_bugs=reported_bugs,
        moderators=moderators,
        banners=banners,
        current_settings=current_settings,
        is_main_admin=is_main_admin,
    )
  return render_template_string(ADMIN_LOGIN_TEMPLATE)


# ==================== DESIGN CSS & TEMPLATES HTML (INTÉGRÉS ET COMPLETS) ====================

COMMON_STYLE = """
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:ital,wght@0,600;0,700;1,400&family=Cinzel:wght@500;700&display=swap" rel="stylesheet">
<style>
  :root {
    --bg-color: #0f0f0f;
    --header-bg: rgba(24, 24, 24, 0.85);
    --text-color: #f1f1f1;
    --text-secondary: #aaa;
    --card-bg: rgba(34, 34, 34, 0.8);
    --border-color: #333;
    --accent-color: #d4af37;
  }
  
  [data-theme="light"] {
    --bg-color: #f8f9fa;
    --header-bg: rgba(255, 255, 255, 0.85);
    --text-color: #1a1a1a;
    --text-secondary: #666;
    --card-bg: rgba(255, 255, 255, 0.85);
    --border-color: #e2e8f0;
    --accent-color: #b89728;
  }

  body { 
    font-family: 'Helvetica Neue', Arial, sans-serif; 
    background: var(--bg-color);
    color: var(--text-color); 
    margin: 0; padding: 0; 
    transition: background 0.3s, color 0.3s; 
    min-height: 100vh;
  }

  header { 
    background: var(--header-bg); 
    backdrop-filter: blur(10px);
    color: var(--accent-color); 
    padding: 12px 18px; 
    display: flex; justify-content: space-between; align-items: center; 
    border-bottom: 1px solid var(--border-color); 
    position: sticky; top: 0; z-index: 1000; 
  }
  
  .brand-container { display: flex; flex-direction: column; align-items: flex-start; }
  .brand-logo { font-family: 'Cinzel', serif; font-size: 26px; font-weight: 700; color: var(--accent-color); letter-spacing: 3px; text-transform: uppercase; margin: 0; }
  .brand-tagline { font-family: 'Playfair Display', serif; font-size: 15px; font-style: italic; color: #f3e5ab; }

  .banner-slider {
    position: relative; width: 100%; height: 180px;
    border-radius: 8px; overflow: hidden; margin: 12px 0;
    border: 1px solid var(--accent-color);
  }
  .banner-slide {
    position: absolute; top: 0; left: 0; width: 100%; height: 100%; opacity: 0; transition: opacity 1s ease-in-out;
  }
  .banner-slide.active { opacity: 1; }
  .banner-slide img { width: 100%; height: 100%; object-fit: cover; }
  .banner-caption {
    position: absolute; bottom: 0; left: 0; right: 0;
    background: linear-gradient(transparent, rgba(0,0,0,0.85));
    color: #f3e5ab; padding: 10px; font-family: 'Playfair Display', serif; font-size: 14px; text-align: center;
  }

  .hamburger-btn { background: none; border: none; color: var(--accent-color); font-size: 26px; cursor: pointer; position: relative; }
  .header-badge { position: absolute; top: -2px; right: -2px; background: #ef4444; color: white; border-radius: 50%; width: 18px; height: 18px; font-size: 11px; display: flex; align-items: center; justify-content: center; }
  
  .nav-menu { 
    display: none; flex-direction: column; position: absolute; top: 100%; right: 0; 
    background: var(--header-bg); width: 260px; border-left: 1px solid var(--border-color);
    box-shadow: 0 10px 25px rgba(0,0,0,0.5); z-index: 999; padding: 8px 0;
  }
  .nav-menu.active { display: flex; }
  .nav-menu a { color: var(--text-color); text-decoration: none; padding: 12px 20px; font-size: 14px; display: flex; justify-content: space-between; }
  .nav-menu a:hover { background: var(--card-bg); color: var(--accent-color); }
  .menu-divider { height: 1px; background: var(--border-color); margin: 6px 15px; }

  .theme-toggle-btn { background: none; border: none; color: var(--text-color); width: 100%; padding: 12px 20px; text-align: left; cursor: pointer; font-weight: bold; }
  
  .container { max-width: 900px; margin: 15px auto; padding: 15px; background: var(--header-bg); border-radius: 8px; border: 1px solid var(--border-color); box-sizing: border-box; }
  
  .btn { background: var(--accent-color); color: #000; padding: 10px 16px; border: none; border-radius: 4px; cursor: pointer; text-decoration: none; font-weight: bold; display: inline-block; text-align: center; }
  .btn:hover { background: #b89728; }
  .btn-back { background: var(--card-bg); color: var(--accent-color); border: 1px solid var(--accent-color); padding: 8px 14px; border-radius: 20px; font-size: 13px; text-decoration: none; display: inline-flex; align-items: center; gap: 6px; }

  .categories { display: flex; gap: 8px; overflow-x: auto; padding-bottom: 8px; scrollbar-width: none; }
  .category-btn { background: var(--card-bg); color: var(--accent-color); border: 1px solid var(--accent-color); padding: 8px 14px; font-size: 12px; border-radius: 20px; text-decoration: none; flex-shrink: 0; }
  .category-btn.active { background-color: var(--accent-color); color: #000; font-weight: 700; }

  .product-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(150px, 1fr)); gap: 12px; }
  .product-card { border: 1px solid var(--border-color); padding: 10px; border-radius: 6px; text-align: center; background: var(--card-bg); display: flex; flex-direction: column; justify-content: space-between; position: relative; }
  
  .star-btn { position: absolute; top: 12px; right: 12px; background: rgba(0,0,0,0.6); border: none; color: #888; font-size: 18px; cursor: pointer; border-radius: 50%; width: 30px; height: 30px; display: flex; align-items: center; justify-content: center; }
  .star-btn.active { color: #ffd700; }

  .alert { padding: 12px; background: #1c1c1c; color: #f3e5ab; margin-bottom: 15px; border-radius: 6px; border: 1px solid var(--accent-color); font-size: 14px; }
  .toast-popup { position: fixed; top: 20px; right: 20px; background: var(--accent-color); color: #000; padding: 14px 20px; border-radius: 8px; font-weight: bold; z-index: 9999; }

  input, select, textarea { background: var(--card-bg); border: 1px solid var(--border-color); color: var(--text-color); padding: 10px; border-radius: 4px; width: 100%; box-sizing: border-box; margin: 5px 0 15px 0; font-size: 14px; }
  table { width: 100%; border-collapse: collapse; margin-top: 15px; display: block; overflow-x: auto; }
  th, td { border: 1px solid var(--border-color); padding: 8px; text-align: left; font-size: 13px; white-space: nowrap; }
  th { background: var(--card-bg); color: var(--accent-color); }
</style>
<script>
  document.addEventListener('DOMContentLoaded', () => {
    const savedTheme = localStorage.getItem('lepa_theme') || 'dark';
    document.documentElement.setAttribute('data-theme', savedTheme);
    updateThemeIcon(savedTheme);
  });
  function toggleTheme() {
    const currentTheme = document.documentElement.getAttribute('data-theme');
    const newTheme = currentTheme === 'light' ? 'dark' : 'light';
    document.documentElement.setAttribute('data-theme', newTheme);
    localStorage.setItem('lepa_theme', newTheme);
    updateThemeIcon(newTheme);
  }
  function updateThemeIcon(theme) {
    const el = document.getElementById('themeIconText');
    if(el) el.innerText = theme === 'light' ? '🌙 Mode Sombre' : '☀️ Mode Clair';
  }
  function showFloatingPopup(message, type = 'success') {
    const popup = document.createElement('div');
    popup.className = 'toast-popup';
    popup.innerHTML = `✨ ${message}`;
    document.body.appendChild(popup);
    setTimeout(() => popup.remove(), 4000);
  }
</script>
"""

NAVIGATION_ITEMS = """
    <button class="theme-toggle-btn" onclick="toggleTheme()"><span id="themeIconText">☀️ Mode Clair</span></button>
    <div class="menu-divider"></div>
    <a href="{{ url_for('index') }}"><span>🏠 Accueil</span></a>
    <a href="{{ url_for('favorites_page') }}"><span>⭐ Mes Favoris</span></a>
    <a href="{{ url_for('index', category='premium') }}"><span>💎 Bijoux Premium</span></a>
    <a href="{{ url_for('cart') }}"><span>🛒 Panier</span></a>
    <a href="{{ url_for('my_orders') }}"><span>📦 Commandes</span></a>
    <div class="menu-divider"></div>
    <a href="{{ url_for('inbox') }}"><span>📬 Boîte de réception</span></a>
    <a href="{{ url_for('suggestions') }}"><span>💡 Suggestions</span></a>
    <a href="{{ url_for('claims') }}"><span>⚠️ Réclamations</span></a>
    <a href="{{ url_for('report_bug') }}"><span>🛠️ Signaler un bug</span></a>
    <a href="{{ url_for('customer_service') }}"><span>🎧 Service Client</span></a>
    <div class="menu-divider"></div>
    {% if session.get('user_name') %}
      <a href="{{ url_for('logout') }}" style="color: #ef4444;"><span>🚪 Déconnexion</span></a>
    {% else %}
      <a href="{{ url_for('login') }}"><span>🔑 Connexion</span></a>
      <a href="{{ url_for('register') }}"><span>📝 Inscription</span></a>
    {% endif %}
"""

HOME_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>LÉPA Joaillerie</title>{COMMON_STYLE}</head><body>
<header><div class="brand-container"><div class="brand-logo">LÉPA</div><div class="brand-tagline">Joaillerie</div></div>
<button class="hamburger-btn" onclick="document.getElementById('hamburgerMenu').classList.toggle('active')">☰</button>
<nav class="nav-menu" id="hamburgerMenu">{NAVIGATION_ITEMS}</nav></header>
<div class="container">
  <div class="categories">
    <a href="{{ url_for('index') }}" class="category-btn {% if not current_category %}active{% endif %}">Tous</a>
    <a href="{{ url_for('index', category='premium') }}" class="category-btn {% if current_category == 'premium' %}active{% endif %}">💎 Premium</a>
  </div>
  <div class="product-grid">
    {{% for p in products %}}
    <div class="product-card">
      <img src="{{{{ p.image if p.image.startswith('http') else url_for('static', filename=p.image) }}}}" style="width:100%; height:110px; object-fit:cover;">
      <h3 style="color:var(--accent-color); font-size:15px;">{{{{ p.name }}}}</h3>
      <p style="font-weight: bold;">{{{{ p.price | format_price }}}} FCFA</p>
      <a href="{{{{ url_for('product_detail', product_id=p.id) }}}}" class="btn" style="padding:5px; font-size:11px;">Voir</a>
    </div>
    {{% endfor %}}
  </div>
</div></body></html>"""

FAVORITES_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Favoris</title>{COMMON_STYLE}</head><body><header><a href="{{{{ url_for('index') }}}}" class="btn-back">← Boutique</a></header><div class="container"><p>Favoris</p></div></body></html>"""
PRODUCT_DETAIL_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Détail</title>{COMMON_STYLE}</head><body><header><a href="{{{{ url_for('index') }}}}" class="btn-back">← Retour</a></header><div class="container"><h2>{{{{ product.name }}}}</h2><p>{{{{ product.price | format_price }}}} FCFA</p><a href="{{{{ url_for('add_to_cart', product_id=product.id) }}}}" class="btn">Ajouter au panier</a></div></body></html>"""
CART_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Panier</title>{COMMON_STYLE}</head><body><header><a href="{{{{ url_for('index') }}}}" class="btn-back">← Boutique</a></header><div class="container"><p>Votre panier</p></div></body></html>"""
CHECKOUT_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Checkout</title>{COMMON_STYLE}</head><body><div class="container"><form method="POST"><input type="text" name="address" placeholder="Adresse" required><input type="number" step="0.1" name="distance" placeholder="Distance km" required><button type="submit" class="btn">Commander</button></form></div></body></html>"""
ORDERS_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Commandes</title>{COMMON_STYLE}</head><body><div class="container"><p>Mes Commandes</p></div></body></html>"""
INBOX_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Inbox</title>{COMMON_STYLE}</head><body><div class="container"><p>Boîte de réception</p></div></body></html>"""
REPORT_BUG_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Bugs</title>{COMMON_STYLE}</head><body><div class="container"><form method="POST"><textarea name="description"></textarea><button type="submit" class="btn">Envoyer</button></form></div></body></html>"""
REGISTER_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Inscription</title>{COMMON_STYLE}</head><body><div class="container"><form method="POST"><input type="text" name="name" required><input type="email" name="email" required><input type="text" name="phone" required><button type="submit" class="btn">S'inscrire</button></form></div></body></html>"""
LOGIN_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Connexion</title>{COMMON_STYLE}</head><body><div class="container"><form method="POST"><input type="email" name="email" required><input type="text" name="phone" required><button type="submit" class="btn">Se connecter</button></form></div></body></html>"""
CUSTOMER_SERVICE_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Support</title>{COMMON_STYLE}</head><body><div class="container"><p>Support Client</p></div></body></html>"""
SUGGESTIONS_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Suggestions</title>{COMMON_STYLE}</head><body><div class="container"><form method="POST"><textarea name="message"></textarea><button type="submit" class="btn">Envoyer</button></form></div></body></html>"""
CLAIMS_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Réclamations</title>{COMMON_STYLE}</head><body><div class="container"><form method="POST"><textarea name="message"></textarea><button type="submit" class="btn">Envoyer</button></form></div></body></html>"""
ADMIN_LOGIN_TEMPLATE = f"""<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Admin Login</title>{COMMON_STYLE}</head><body><div class="container"><form method="POST"><input type="password" name="code" required><button type="submit" class="btn">Connexion</button></form></div></body></html>"""

ADMIN_TEMPLATE = f"""
<!DOCTYPE html><html lang="fr"><head><meta charset="UTF-8"><title>Admin</title>{COMMON_STYLE}</head>
<body><header><h1>Panneau Admin</h1><a href="{{{{ url_for('index') }}}}" class="btn-back">Voir le site</a></header>
<div class="container">
  <div style="background:var(--card-bg); padding:15px; border-radius:8px;">
    <h3>Statistiques</h3>
    <p>Clients: {{{{ users_count }}}}</p>
    <p>Commandes: {{{{ orders | length }}}}</p>
  </div>
</div></body></html>
"""

# --- LANCEMENT DE L'APPLICATION (MODE LOCAL OU PRODUCTION WSGI) ---
if __name__ == "__main__":
  init_db()
  port = int(os.environ.get("PORT", 5000))
  app.run(host="0.0.0.0", port=port, debug=False)
else:
  # Initialisation automatique de la base lors du déploiement en ligne (ex: Gunicorn)
  init_db()

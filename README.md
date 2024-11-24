from flask import Flask, render_template, request, redirect, url_for, session
from flask_sqlalchemy import SQLAlchemy
from werkzeug.security import generate_password_hash, check_password_hash
import datetime

app = Flask(__name__)
app.secret_key = 'your_secret_key'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///app.db'
db = SQLAlchemy(app)


class Customer(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), nullable=False, unique=True)
    password = db.Column(db.String(100), nullable=False)
    package_expiry = db.Column(db.DateTime, nullable=False)
    blacklisted = db.Column(db.Boolean, default=False)
    renewed = db.Column(db.Boolean, default=False)

class Admin(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(50), nullable=False, unique=True)
    password = db.Column(db.String(100), nullable=False)


class BlacklistNotification(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    customer_id = db.Column(db.Integer, db.ForeignKey('customer.id'))
    notification_date = db.Column(db.DateTime, default=datetime.datetime.utcnow)
    content = db.Column(db.String(255), nullable=False)


class BlacklistAppeal(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    customer_id = db.Column(db.Integer, db.ForeignKey('customer.id'))
    appeal_date = db.Column(db.DateTime, default=datetime.datetime.utcnow)
    content = db.Column(db.String(255), nullable=False)
    status = db.Column(db.String(20), default='Pending')  # 'Pending', 'Approved', 'Denied'

class Contract(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    customer_id = db.Column(db.Integer, db.ForeignKey('customer.id'))
    start_date = db.Column(db.DateTime, default=datetime.datetime.utcnow)
    end_date = db.Column(db.DateTime, nullable=False)
    renewed = db.Column(db.Boolean, default=False)

@app.route('/customer/login', methods=['GET', 'POST'])
def customer_login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        customer = Customer.query.filter_by(username=username).first()
        if customer and check_password_hash(customer.password, password):
            session['customer_id'] = customer.id
            return redirect(url_for('customer_dashboard'))
    return render_template('customer_login.html')

@app.route('/customer/dashboard')
def customer_dashboard():
    if 'customer_id' not in session:
        return redirect(url_for('customer_login'))
    customer = Customer.query.get(session['customer_id'])
    contract = Contract.query.filter_by(customer_id=customer.id).first()
    appeal = BlacklistAppeal.query.filter_by(customer_id=customer.id, status='Pending').first()
    return render_template('customer_dashboard.html', customer=customer, contract=contract, appeal=appeal)


@app.route('/customer/renew', methods=['POST'])
def renew():
    customer = Customer.query.get(session['customer_id'])
    if not customer.renewed:
        new_end_date = customer.package_expiry + datetime.timedelta(days=30)  # 续约30天
        contract = Contract(customer_id=customer.id, end_date=new_end_date, renewed=True)
        db.session.add(contract)
        customer.renewed = True
        customer.package_expiry = new_end_date
        db.session.commit()
        return redirect(url_for('customer_dashboard'))
    return "You have already renewed."


@app.route('/customer/appeal_blacklist', methods=['POST'])
def appeal_blacklist():
    if 'customer_id' not in session:
        return redirect(url_for('customer_login'))

    customer = Customer.query.get(session['customer_id'])
    
    if not customer.blacklisted:
        return redirect(url_for('customer_dashboard'))
    
    content = request.form['content']  # 顾客申请解除黑名单的理由
    appeal = BlacklistAppeal(customer_id=customer.id, content=content)
    db.session.add(appeal)
    db.session.commit()
    
    return redirect(url_for('customer_dashboard'))


@app.route('/admin/login', methods=['GET', 'POST'])
def admin_login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        admin = Admin.query.filter_by(username=username).first()
        if admin and check_password_hash(admin.password, password):
            session['admin_id'] = admin.id
            return redirect(url_for('admin_dashboard'))
    return render_template('admin_login.html')

@app.route('/admin/dashboard')
def admin_dashboard():
    if 'admin_id' not in session:
        return redirect(url_for('admin_login'))
    customers = Customer.query.all()
    return render_template('admin_dashboard.html', customers=customers)
@app.route('/admin/blacklist/<int:id>')
def blacklist(id):
    customer = Customer.query.get(id)
    customer.blacklisted = True
    db.session.commit()

    notification = BlacklistNotification(customer_id=customer.id, content="You have been blacklisted.")
    db.session.add(notification)
    db.session.commit()

    return redirect(url_for('admin_dashboard'))

@app.route('/admin/unblacklist/<int:id>')
def unblacklist(id):
    customer = Customer.query.get(id)
    customer.blacklisted = False
    db.session.commit()
    return redirect(url_for('admin_dashboard'))

@app.route('/admin/blacklist_appeals')
def blacklist_appeals():
    if 'admin_id' not in session:
        return redirect(url_for('admin_login'))
    
    appeals = BlacklistAppeal.query.filter_by(status='Pending').all()
    return render_template('admin_blacklist_appeals.html', appeals=appeals)

@app.route('/admin/approve_appeal/<int:id>')
def approve_appeal(id):
    appeal = BlacklistAppeal.query.get(id)
    appeal.status = 'Approved'
    
    customer = Customer.query.get(appeal.customer_id)
    customer.blacklisted = False
    
    db.session.commit()
    return redirect(url_for('blacklist_appeals'))

@app.route('/admin/deny_

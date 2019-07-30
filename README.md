# flask-bp
Boilerplate for Flask

Pre-requirement:
  
  python 3.7.4
  pipenv
 
 Step1: Create project directory named "flask-bf"

          mkdir flask-bf
          cd flask-bf
 Step2: Install flask using pipenv
        
        pipenv shell
        pipenv install flask
 Now the directory looks like

.
  ├── Pipfile
  └── Pipfile.lock
Step3: Create a package called "app" inside flask-bp directory
      
      mkdir app
      cd app
Step4: Inside app directory create a file named __init__.py and save the below code. Next you have to change the url_prefix value '/boilerplate/v1' as required in the endpoints.

    # flask-bp/app/__init__.py
    from flask import Flask
    from flask_sqlalchemy import SQLAlchemy
    from flask_migrate import Migrate
    from config import Config
    from flask_cors import CORS

    db = SQLAlchemy()
    migrate = Migrate()


    def create_app(config_class=Config):
        app = Flask(__name__)
        app.config.from_object(config_class)
        CORS(app)

        db.init_app(app)
        migrate.init_app(app, db)

        from app.api import bp as api_bp
        app.register_blueprint(api_bp, url_prefix='/boilerplate/v1')

        return app

    from app import models

Step5: Install the packages which are used in step4.
        
        pipenv install flask-sqlalchemy
        pipenv install flask-migrate
        pipenv install Migrate
        pipenv install flask-cors
        pipenv install python-dotenv
        pipenv install Flask-HTTPAuth

 Step6: Create "config.py" in root directory. To store the configuration datas.
    
            cd ..
            touch config.py
Append the below mentioned code into the config.py file.
          
      ~/flask-bp/config.py
      import os
      from dotenv import load_dotenv

      basedir = os.path.abspath(os.path.dirname(__file__))
      load_dotenv(os.path.join(basedir, '.env'))


      class Config(object):
          DEBUG = os.environ.get('DEBUG') or False
          SECRET_KEY = os.environ.get('SECRET_KEY') or "ACSGSUGQJKDQTVDVQIIUDVBCREQVXPOOHKBEQMNFEIKVXH"
          PASSWORD_RESET_SALT = os.environ.get('PASSWORD_RESET_SALT') or "GFVDHWBIJBCWHBCHBIHCJJWccFWEF"
          PORT = os.environ.get('PORT') or 5000
          SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
              'sqlite:///' + os.path.join(basedir, 'app.db')
          SQLALCHEMY_TRACK_MODIFICATIONS = False
          PER_PAGE = os.environ.get('PER_PAGE')
          # S3_BUCKET = os.environ.get('S3_BUCKET')
          # AWS_ACCESS_KEY_ID = os.environ.get('AWS_ACCESS_KEY_ID')
          # AWS_SECRET_ACCESS_KEY = os.environ.get('AWS_SECRET_ACCESS_KEY')
          # SENDGRID_API_KEY = os.environ.get('SENDGRID_API_KEY')
          # STRIPE_SECRET_KEY = os.environ.get('STRIPE_SECRET_KEY')
 
 Step7: Create a file runserver.py in the root directory and append the code. Import all models and                                 return in make_shell_context() 
 
        ~/flask-bp/runserver.py
        
        from app import create_app, db
        from app.models import Users, 

        app = create_app()


        @app.shell_context_processor
        def make_shell_context():
            return {'db': db, 'Users': Users, 'key': 'value'}


        if __name__ == "__main__":
            # import ptvsd
            # address = ('0.0.0.0', 3000)
            # ptvsd.enable_attach(address)
            app.run(host='0.0.0.0', debug=app.config["DEBUG"], port=app.config["PORT"])
            
  Now the directory looks like:
  
      .
    ├── app
    │   └── __init__.py
    ├── config.py
    ├── Pipfile
    ├── Pipfile.lock
    └── runserver.py
Step10: Make three folders inside the folder "app", "models" to add all models, "services" to add all logical side functions and "api" to have all api routes in single folder.
        
        cd app/
        mkdir api
        mkdir models
        mkdir services
 Now the directory looks like:
 
    ├── app
    │   ├── api
    │   ├── __init__.py
    │   ├── models
    │   └── services
    ├── config.py
    ├── Pipfile
    ├── Pipfile.lock
    └── runserver.py

Step11: Inside these 3 folders create an empty file "__init__.py". Later all the files inside each folders should be added in the corresponding __init__.py file. Create another folder "templates" to store template files, here __init__.py is not required.

Now the directory looks like:

    .
    ├── app
    │   ├── api
    │   │   └── __init__.py
    │   ├── __init__.py
    │   ├── models
    │   │   └── __init__.py
    │   └── services
    │   │  └── __init__.py
    │   └── templates
    ├── config.py
    ├── Pipfile
    ├── Pipfile.lock
    └── runserver.py

Step12: Next lets create models.
        
        cd models
        
First create a file "base.py" which can be used by any other models. Inside the base class default created and updated columns are added, get all datas from a model and return the result in pagination form as dict.

    # ~/flask-bp/app/models/base.py
    
    from app import db
    from config import Config


    class BaseModel(db.Model):
        __abstract__ = True
        created = db.Column(db.DateTime, default=db.func.now())
        updated = db.Column(db.DateTime, default=db.func.now(), onupdate=db.func.now())

        @classmethod
        def to_collection_dict(cls, url, page=1, per_page=20, **kwargs):
            resources = cls.query.paginate(page, per_page, False)
            data = {
                'item': [item.to_dict() for item in resources.items],
                '_meta': {
                    'page': page,
                    'per_page': per_page,
                    'total_pages': resources.pages,
                    'total_items': resources.total,
                    'url': Config.BASE_URL + url
                }

            }
            return {"status": 200, "message": "success", "data": data.get('item', ""), "_meta": data.get('_meta', "")}, True

        def to_dict(self):
            pass


Next create a users model "users.py" inside the directory models itself
    
    # ~/flask-bp/app/models/users.py
    from app import db
    from config import Config
    from werkzeug.security import generate_password_hash, check_password_hash
    from flask_httpauth import HTTPBasicAuth
    from itsdangerous import (TimedJSONWebSignatureSerializer
                              as Serializer, BadSignature, SignatureExpired)
    from app.models import BaseModel
    from app.constants import Access

    auth = HTTPBasicAuth()

    class Users(BaseModel):
        __tablename__ = 'users'
        id = db.Column(db.Integer, primary_key=True, autoincrement=True)
        email = db.Column(db.String(128), index=True, unique=True)
        first_name = db.Column(db.String(128))
        last_name = db.Column(db.String(64))
        phone = db.Column(db.String(15))
        password = db.Column(db.String(128))
        auth_token = db.Column(db.String(300))
        #address1 = db.Column(db.String(128))
        #address2 = db.Column(db.String(128))
        #state = db.Column(db.String(64))
        #country = db.Column(db.String(64))
        #zipcode = db.Column(db.Integer)
        #status = db.Column(db.Boolean)
        #role_id = db.Column(db.Integer, db.ForeignKey('roles.id'))
        #token_expiration = db.Column(db.String(128))
        #orders = db.relationship('Orders', backref='user_order', lazy='dynamic')
        #transactions = db.relationship('Transactions', backref='user_transaction', lazy='dynamic')

        def to_dict(self):
            data = {
                'id': self.id,
                'user_id': self.id,
                'email': self.email,
                'first_name': self.first_name,
                'last_name': self.last_name,
                'name': ((self.first_name or ' ') + " " + (self.last_name or ' ')).strip(),
                'phone': self.phone,
                #'address': ((self.address1 or '') + ", " + (self.address2 or ' ')).strip(),
                #'country': self.country,
                #'zip': self.zipcode,
                #'role_id': self.role_id,
                'created': self.created.strftime('%Y-%m-%d %H:%M %p')
            }
            return data

        def get_hashed_password(self, password):
            return generate_password_hash(password)

        def hash_password(self, password):
            self.password = generate_password_hash(password)

        def check_password(self, password):
            return check_password_hash(self.password, password)

        def generate_auth_token(self, expiration=600):
            s = Serializer(Config.SECRET_KEY, expires_in=expiration)
            return s.dumps({'id': self.id})

        def is_admin(self):
            return self.role_id == Access.ADMIN

        @staticmethod
        def verify_auth_token(token):
            s = Serializer(Config.SECRET_KEY)
            try:
                data = s.loads(token)
            except SignatureExpired:
                return None
            except BadSignature:
                return None
            user = Users.query.get(data['id'])
            return user
  
 Step13: Now we had 2 models so next we need to add this in __init__.py of models.
 
            #~/flask-bp/app/models/__init__.py
            from app.models.base import BaseModel
            from app.models.users import Users
            
        Like this if we create new models then update it in the __init__.py file also.
 Step14: 
 

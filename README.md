*The character ~ means that the following is a console command.

Pre
1. Install new app.
	~ sails new appname
2. Change directory into the app
	~ cd appname

**************************************
*	Went along with blog
*	https://medium.com/@greg.hesp/implement-passport-js-authentication-with-sails-js-1-0-50888265fb83
**************************************
	
1. install the npm packages
	~ npm install passport
	~ npm install passport-local
	~ npm install bcrypt-nodejs
	
2. Add the following to the middleware for the passport application in "config/http.js",
   2.1 First uncomment order
   2.2 Insert "passportInit" and "passportSession" below "session".

   order: [
      'cookieParser',
      'session',
      'passportInit',            // <==== If you're using "passport", you'll want to have its two
      'passportSession',         // <==== middleware functions run after "session".
      'bodyParser',
      'compress',
      'poweredBy',
      'router',
      'www',
      'favicon',
    ],
	
3. Next, Add middleware initialisers for passport and passport-local to "config/http.js"
	
	/***************************************************************************
    *                                                                          *
    * Initialise for both passport and passport-local                          *
    *                                                                          *
    * https://sailsjs.com/config/http#?customizing-the-body-parser             *
    *                                                                          *
    ***************************************************************************/
    
    passportInit    : (function (){
      return require('passport').initialize();
    })(),

    passportSession : (function (){
      return require('passport').session();
    })()
	
	
	
4. Create a user model
	4.1 Use the following command to create the model
		~ sails generate model user
	
	4.2 Add the following to the newly created user model.

	const bcrypt = require('bcrypt-nodejs');
	module.exports = {
	attributes: {
		email: {
		  type: 'email',
		  required: true,
		  unique: true
		},
		username: {
		  type: 'string',
		  required: true,
		  unique: true
		},
		password: {
		  type: 'string',
		  required: true
		}
	  },
	  customToJSON: function() {
		 return _.omit(this, ['password'])
	  },
	  beforeCreate: function(user, cb){
		bcrypt.genSalt(10, function(err, salt){
		  bcrypt.hash(user.password, salt, null, function(err, hash){
			if(err) return cb(err);
			user.password = hash;
			return cb();
		  });
		});
	  }
	};
	
5. Create an Auth controller
	5.1 Use the following command to create the controller
		~ sails generate controller auth
		
	5.2 Add the following to the Auth controller
	
	/**
	 * AuthController
	 *
	 * @description :: Server-side actions for handling incoming requests.
	 * @help        :: See https://sailsjs.com/docs/concepts/actions
	 */
	const passport = require('passport');
	module.exports = {
		//Login function
		login: function(req, res) {
			passport.authenticate('local', function(err, user, info){
				if((err) || (!user)) return res.send({ message: info.message, user});
				
				req.login(user, function(err) {
					if(err) res.send(err);
					return res.redirect('/');
				});
			})(req, res);
		},
		//Logout function
		logout: function(req, res) {
			req.logout();
			res.redirect('/');
		},
		//Register function
		register: function(req, res){
			//TODO: form validation here
			data = {
				username: req.body.username,
				email: req.body.email,
				password: req.body.password,
				description: req.body.description
			}
			User.create(data).fetch().exec(function(err, user){
				if (err) return res.negoiate(err);

				//TODO: Maybe send confirmation email to the user before login
				req.login(user, function(err){
					if (err) return res.negotiate(err);
					sails.log('User '+ user.id +' has logged in.');
					return res.redirect('/');
				})
			})
		}
	};
	
6. Create a new file in the config folder named 'passport.js', Load passport via require and bcrypt for password hashing.
	
	const passport = require('passport'),
	LocalStrategy = require('passport-local').Strategy,
	bcrypt = require('bcrypt-nodejs');
	passport.serializeUser(function(user, cb) {
		cb(null, user.id);
	});
	passport.deserializeUser(function(id, cb){
		User.findOne({id}, function(err, user) {
			cb(err, user);
		});
	});
	passport.use(new LocalStrategy({
			usernameField: 'username',
			passportField: 'password'
		}, function(username, password, cb){
		User.findOne({username: username}, function(err, user){
			if(err) return cb(err);
			if(!user) return cb(null, false, {message: 'Username not found'});
			bcrypt.compare(password, user.password, function(err, res){
				if(!res) return cb(null, false, { message: 'Invalid Password' });
				return cb(null, user, { message: 'Login Succesful'});
			});
		});
	}));

7. Add the following policies "config/policies.js" which will block all controllers except the Auth controller from access before signing in.
	/***************************************************************************
	*                                                                          *
	* Default policy for all controllers and actions, unless overridden.       *
	* (`true` allows public access)                                            *
	*                                                                          *
	***************************************************************************/

	'*': 'authenticated',
	// whitelist the auth controller
	'auth': {
		'*': true
	}
	
8. Add the mongodb url and adapter to the default settings page "config/datastores.js"
	adapter: 'sails-mongo',
	url: 'mongodb://localhost:27017/passportexample',
	
9. Update the models config file "config/models.js" with the different id field when using mongodb.
	attributes: {
		createdAt: { type: 'number', autoCreatedAt: true, },
		updatedAt: { type: 'number', autoUpdatedAt: true, },
		id: { type: 'string', columnName: '_id' },
		//--------------------------------------------------------------------------
		//  /\   Using MongoDB?
		//  ||   Replace `id` above with this instead:
		//
		// ```
		// id: { type: 'string', columnName: '_id' },
		// ```
		//--------------------------------------------------------------------------
	},
	
10. Make sure to add your urls to the routes page "config/routes.js"
	//Auth
	'get /login': {
		view: 'user/login'
	},
	'get /register': {
		view: 'user/register'
	},
	'post /login': 'AuthController.login',
	'post /register': 'AuthController.register',
	'/logout': 'AuthController.logout',
	
11. The HTML forms representing the Register page and Login Page.
	11.1 Register Page
	<h1>Signup</h1>
	<form action="/register" method="post">
		<div class="form-group">
			<label for="username">Choose a Username</label>
			<input name="username" class="form-control" type="text" placeholder="@username" />
		</div>

		<div class="form-group">
			<label for="email">Primary Email</label>
			<input name="email" class="form-control" type="email"/>
		</div>

		<div class="form-group">
			<label for="description">Description</label>
			<textarea name="description" placeholder="A little about your self"></textarea>
		</div>

		<label for="password">Choose a Password</label>
		<input name="password" class="form-control" type="password"/>

		<input class="btn btn-primary" type="submit" value="Register"/>
	</form>
	
	11.2 Login Page
	<h1>Login</h1>
	<form action="/login" method="post">
	  <div class="form-group">
		<label for="username">Username</label>
		<input name="username" type="text" class="form-control"/>
	  </div>
	  <div class="form-group">
		<label for="password">Password</label>
		<input name="password" type="password" class="form-control"/>
	  </div>

	  <input class="btn btn-primary" type="submit" value="Login" />
	</form>
  
12. Optional Step: Add Session Store into MongoDB, Add the following into "config/session.js"

	adapter: 'connect-mongo',
	// Note: user, pass and port are optional
	url: 'mongodb://localhost:27017/passportexample',
	collection: 'sessions',
	auto_reconnect: false,
	ssl: false,
	stringify: true
	
	
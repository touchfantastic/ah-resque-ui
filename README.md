# AH-RESQUE-UI
A resque administration website for actionhero

## Setup

- `npm install --save ah-resque-ui`
- `npm run actionhero -- link --name ah-resque-ui`
- set up the required routes in your project

```js
exports.default = {
  routes: function(api){
    return {
      get: [
        { path: '/resque/packageDetails',       action: 'resque:packageDetails'    },
        { path: '/resque/resqueDetails',        action: 'resque:resqueDetails'     },
        { path: '/resque/loadWorkerQueues',     action: 'resque:loadWorkerQueues'  },
        { path: '/resque/resqueFailedCount',    action: 'resque:resqueFailedCount' },
        { path: '/resque/resqueFailed',         action: 'resque:resqueFailed'      },
        { path: '/resque/delayedjobs',          action: 'resque:delayedjobs'       },
      ],

      post: [
        { path: '/resque/removeFailed',            action: 'resque:removeFailed'            },
        { path: '/resque/retryAndRemoveFailed',    action: 'resque:retryAndRemoveFailed'    },
        { path: '/resque/removeAllFailed',         action: 'resque:removeAllFailed'         },
        { path: '/resque/retryAndRemoveAllFailed', action: 'resque:retryAndRemoveAllFailed' },
        { path: '/resque/forceCleanWorker',        action: 'resque:forceCleanWorker'        },
        { path: '/resque/delDelayed',              action: 'resque:delDelayed'              },
        { path: '/resque/runDelayed',              action: 'resque:runDelayed'              },
      ]
    }
  }
};
```

## Authentication Via Middleware
This package exposes some potentially dangerous actions which would allow folks to see user data (if you keep such in your task params), and modify your task queues.  To protect these actions, you should configure this package to use [action middleware](http://www.actionherojs.com/docs/#action-middleware) which would restrict these actions to only certain clients.

For example, you might have roles (`admin`, `analyst`, etc), and require that a certain session (determined via fingerprint) to be pre-logged in.  In this case, you might have an initializer like the following:

```js
// from initializers/session

module.exports = {
  initialize: function (api, next) {

    var redis = api.redis.clients.client;

    api.session = {
      prefix: 'session:',
      ttl: 60 * 60 * 24, // 1 day

      load: function(connection, callback){
        var key = api.session.prefix + connection.fingerprint;

        redis.get(key, function(error, data){
          if(error){     return callback(error);       }
          else if(data){ return callback(null, JSON.parse(data));  }
          else{          return callback(null, false); }
        });
      },

      create: function(connection, user, callback){
        var key = api.session.prefix + connection.fingerprint;

        var sessionData = {
          userId:          user.id,
          status:          user.status,
          sesionCreatedAt: new Date().getTime()
        };

        redis.set(key, JSON.stringify(sessionData), function(error, data){
          if(error){ return callback(error); }
          redis.expire(key, api.session.ttl, function(error){
            callback(error, sessionData);
          });
        });
      },

      middleware: {
        // These actions are restricted to the website (and you need a CSRF token)
        'logged-in-session': {
          name: 'logged-in-session',
          global: false,
          priority: 1000,
          preProcessor: function(data, callback){
            api.session.load(data.connection, function(error, sessionData){
              if(error){ return callback(error); }
              else if(!sessionData){
                return callback(new Error('Please log in to continue'));
              }else{
                data.session = sessionData;
                var key = api.session.prefix + data.connection.fingerprint;
                redis.expire(key, api.session.ttl, callback);
              }
            });
          }
        }

      }
    };

    api.actions.addMiddleware(api.session.middleware['logged-in-session']);

    next();
  }
};
```

Now you can apply the `logged-in-session` middleware to your actions to protect them.  

To inform ah-resque-ui to use a middleware determined elsewhere like this, set `api.config.ah-resque-ui.middleware = 'logged-in-session'` 
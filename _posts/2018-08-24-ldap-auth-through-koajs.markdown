---
title: How to create Promise over LDAP auth on KoaJS
layout: post
date: 2018-08-24 16:00
tag:
- coding, koajs, nodejs, middleware, promise, ldap, authetication
category: thoughts
author: alexghirelli
description: A few days during the implementation of a middleware that allows the authentication of users through the LDAP protocol, I found myself faced with... 
---

![Markdowm Image][koa]{: class="bigger-image" }

A few days during the implementation of a middleware that allows the authentication of users through the **LDAP** protocol, I found myself faced with a major problem: the module "**ldapauth-fork**" that I used is not compatible with the **Promise**, and this does not allow the proper use of data.

Below I leave you the solution that I found by doing various tests with the module "**bluebird**".

As a first step we will have to create a "router" in which we will define the configuration of our API.

{% highlight html %}

import 'babel-polyfill';
import Router from 'koa-router';
import { baseApi } from '../../config/env/common';
const {login} = require('../controllers/auth')

const api = 'authenticate';

const router = new Router();

router.prefix(`/${baseApi}/${api}`);

// POST /api/authenticate
router.post('/', login, (req, res, next) => {});

export default router;

{% endhighlight %}

Now we create our middleware that will call the LDAP services and authenticate and export it as a Node module.

{% highlight html %}
import LdapAuth from 'ldapauth-fork';

const ldap = new LdapAuth({
  url: 'ldaps://...',
  bindDN: 'cn=Manager,dc=com',
  bindCredentials: 'secret',
  searchBase: 'ou=people,dc=com',
  searchFilter: '(uid={{username}})'
})

ldap.on('error', (err) => { throw err })

module.exports = ldap
{% endhighlight %}

Now we'll just need a controller inside which we'll take the data passed into the POST call and pass it to our Promise. As you can see we call the method "authenticate" inside the Promise in order to steal the data related to the user's profile and save it in a constant "profile".

{% highlight html %}

import jwt from 'jsonwebtoken';
import ldap from '../ldap';
import Promise from 'bluebird';

exports.login = async (req, res) => {
  const { username, password } = req.body

  if (!username || !password) {
    res.status(400
    res.json({ error: 'Missing username or password.' })
    return
  }

  const profile = await Promise.fromCallback(cb => ldap.authenticate(username, password, cb))
  const token = jwt.sign({ user: profile }, 'secret', {})

  const { exp } = jwt.verify(token, 'secret')

  res.json({
    access_token: token,
    token_type: 'Bearer',
    expires_in: exp,
    user: profile
  })
}

{% endhighlight %}

From this point on with the content of the constant "profile" we can do what we want: generate a BEARER token or read its content, parse it and call us more methods above and then return it all to the response of our API.

[main]: http://blog.builtinnode.com/uploads/covers/53E787AE-3AF7-11E7-BCD4-AC39D20E7A71.
[koa]: https://codecondo.com/wp-content/uploads/2015/10/Koa.js.jpg

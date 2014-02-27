#!/usr/bin/python
# xname-dyndns - Dynamic Dns client for xname.org
# Copyright (C) 2014 Yann Rouillard <yann@pleides.fr.eu.org>

# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 2 of the License, or
# (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA 02111-1307 USA.
#

from __future__ import print_function

import xmlrpclib
import argparse
import dns.resolver
import requests
import logging as log
import sys
import os


#############################################################################
# Useful functions
#############################################################################

def get_resolved_ip(server, nameservers_domain='xname.org'):
    """ Return the list of IP addresses for the given server
        according to the name servers of the domaine
        nameservers_domain.

        The nameservers of the given domain will be directly used
        to retrieve that information instead of the default resolver
    """
    resolver = dns.resolver.Resolver()

    # We first retrieve the IP of xname.org nameservers to
    # be able to use them directly for name resolution
    query_answers = resolver.query(nameservers_domain, 'NS')
    nameservers = []
    for answer in query_answers:
        name = answer.to_text().rstrip('.')
        nameservers.extend([x.to_text() for x in resolver.query(name, 'A')])

    resolver.nameservers = nameservers

    ip = []

    # We try nameservers one by one to workaround a bug
    # with dns python, see https://github.com/rthalley/dnspython/issues/31
    for nameserver in nameservers:
        resolver.nameservers = [nameserver]

        try:
            answer = resolver.query(server, 'A')
            ip = [x.to_text() for x in answer]
            break

        except dns.resolver.NoAnswer:
            # NoAnswer means that the name has not yet been
            # registered on xname.org
            break
        except NameError as e:
            # Special case to workaround dns python bug
            if 'retry_servfail' in e.message:
                continue
            else:
                raise

    return ip


def get_external_ip(resolver_url='http://icanhazip.com'):
    """ Return the external ip address of the machine
        executing the current code using the resolver service
        available at the url resolver_url
    """
    r = requests.get(resolver_url)

    if r.status_code != 200:
        raise requests.exceptions.HTTPError()

    return r.text.strip()

#############################################################################
# Main program
#############################################################################

# Default values
default_xname_url = "https://xname.org/xmlrpc.php"
default_ttl = '600'

xmlrpc_param_names = ('user', 'password', 'ttl', 'zone', 'name')

parser = argparse.ArgumentParser(description='Xname.org dynamic dns updater')
parser.add_argument('name', metavar='HOSTNAME',
                    help='update DNS information for this host')
parser.add_argument('--user', dest='user',
                    help='login as user on xname.org')
parser.add_argument('--password', dest='password',
                    help='use password to login on xname.org')
parser.add_argument('--ttl', dest='ttl', default=default_ttl,
                    help='Time To Live value to set for the dns entry')
parser.add_argument('--zone', dest='zone',
                    help='name of the zone')
parser.add_argument('--xname_url', dest='xname_url', default=default_xname_url,
                    help='url of the xname.org XMLRPC service')
parser.add_argument('--verbose', dest='verbose', action='store_true',
                    help='turn on verbose output')
parser.add_argument('--debug', dest='debug', action='store_true',
                    help='turn on debug output')
args = parser.parse_args()


log.basicConfig(format="%(levelname)s: %(message)s")
if args.debug:
    log.getLogger().setLevel(level=log.DEBUG)
elif args.verbose:
    log.getLogger().setLevel(level=log.INFO)
else:
    log.getLogger().setLevel(level=log.WARNING)


if not args.user and os.getenv('XNAME_USER'):
    args.user = os.getenv('XNAME_USER')

if not args.password and os.getenv('XNAME_PASSWORD'):
    args.password = os.getenv('XNAME_PASSWORD')

if not args.user and not args.password:
    log.error('you must specify a user and a password using the'
              ' command line\n       or by setting environment variables'
              ' XNAME_USER and XNAME_PASSWORD.')
    sys.exit(2)


external_ip = get_external_ip()
log.info('External IP address is %s' % external_ip)
resolved_ip = get_resolved_ip(args.name)
log.info('Currently resolved IP addresses are %s' %
         (', '.join(resolved_ip) if resolved_ip else '(none)'))


if len(resolved_ip) == 1 and external_ip == resolved_ip[0]:
    log.info('IP address did not change. Exiting.')

else:
    log.info('Updating Xname.org DNS information for %s' % args.name)

    client = xmlrpclib.Server(args.xname_url)

    xmlrpc_params = {name: getattr(args, name) for name in xmlrpc_param_names
                     if name in args}

    # We guess the zone name from the hostname if we were given
    # a fully qualified one
    if not xmlrpc_params['zone'] and '.' in xmlrpc_params['name']:
        xmlrpc_params['name'], xmlrpc_params['zone'] = args.name.split('.', 1)

    # oldaddress = * means that all previous IP registered will be replaced
    # with the new one
    xmlrpc_params['oldaddress'] = '*'
    xmlrpc_params['newaddress'] = external_ip

    log.debug('XML-RPC parameters: %s' % xmlrpc_params)
    result = client.xname.updateArecord(xmlrpc_params)
    log.debug('XML-RPC return: %s' % result)

    if 'faultString' in result:
        log.error("Can't update DNS information: %s" % result['faultString'])
        sys.exit(3)

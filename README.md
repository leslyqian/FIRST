# 创建Python第三方库——DLT645通讯协议
***
## pydlt645 Client written in Python-2.7. 
* [IDE](#ide)
* [Get Source Code Of pymodbus](#get-source-code-of-pymodbus)
* [Create Source Code Of pydlt645](#create-source-code-of-pydlt645)
* [Debug Of pydlt645](#debug-of-pydlt645)
* [Install、Uninstall And Test Of pydlt645](#install-uninstall-and-test-of-pydlt645)

## IDE
### Installation of Eclipse
[Download](https://eclipse.org/downloads/eclipse-packages/)
![p1](http://i.imgur.com/P4iRhIl.png)
### Installation of Eclipse plug-ins
1、Run Eclipse by double clicking **eclipse.exe** in the Eclipse installation directory.
2、Select menu command **Help | Eclipse** Marketplace. 
3、In a dialog type search string **'pydev'** and click go.   
![p2](http://i.imgur.com/crLWFma.png)      
                                                                                                          
## Get Source Code Of pymodbus
[Download](https://github.com/bashwork/pymodbus)

## Create Source Code Of pydlt645(reference pymodbus)
1、Create python project in eclipse IDE.
File|New|Project -> PyDev Project
2、Create the following folders and files.
![p3](http://i.imgur.com/f5PaZSQ.png)
**Code:**
pydlt645.\__init__.py
```ruby
'''
Pydlt645: Dlt645 Protocol Implementation
'''

import pydlt645.version as __version
__version__ = __version.version.short()
__author__  = 'qian qin'

#---------------------------------------------------------------------------#
# Block unhandled logging
#---------------------------------------------------------------------------#
import logging as __logging
try:
    from logging import NullHandler as __null
except ImportError:
    class __null(__logging.Handler):
        def emit(self, record):
            pass

__logging.getLogger(__name__).addHandler(__null())

#---------------------------------------------------------------------------#
# Define True and False if we don't have them (2.3.2)
#---------------------------------------------------------------------------#
try:
    True, False
except NameError:
    True, False = (1 == 1), (0 == 1)
```
pydlt645.constants.py
```ruby
'''
Constants For Dlt645 Server/Client
----------------------------------

This is the single location for storing default
values for the servers and clients.
'''
from pydlt645.interfaces import Singleton


class Defaults(Singleton):
    ''' A collection of dlt645 default value

    .. attribute:: Retries

       The default number of times a client should retry the given
       request before failing (3)

    .. attribute:: RetryOnEmpty

       A flag indicating if a transaction should be retried in the
       case that an empty response is received. This is useful for
       slow clients that may need more time to process a requst.

    .. attribute:: Timeout

       The default amount of time a client should wait for a request
       to be processed (3 seconds)

    .. attribute:: TransactionId

       The starting transaction identifier number (0)

    .. attribute:: ProtocolId

       The dlt645 protocol id.  Currently this is set to 0 in all
       but proprietary implementations.

    .. attribute:: Baudrate

       The speed at which the data is transmitted over the serial line.
       This defaults to 19200.

    .. attribute:: Parity

       The type of checksum to use to verify data integrity. This can be
       on of the following::

         - (E)ven - 1 0 1 0 | P(0)
         - (O)dd  - 1 0 1 0 | P(1)
         - (N)one - 1 0 1 0 | no parity

       This defaults to (N)one.

    .. attribute:: Bytesize

       The number of bits in a byte of serial data.  This can be one of
       5, 6, 7, or 8. This defaults to 8.

    .. attribute:: Stopbits

       The number of bits sent after each character in a message to
       indicate the end of the byte.  This defaults to 1.

    '''
    Retries             = 3
    RetryOnEmpty        = False
    Timeout             = 3
    TransactionId       = 0
    ProtocolId          = 0
    UnitId              = 0x00
    Baudrate            = 19200
    Parity              = 'N'
    Bytesize            = 8
    Stopbits            = 1

#---------------------------------------------------------------------------#
# Exported Identifiers
#---------------------------------------------------------------------------#
__all__ = ["Defaults"]
```
pydlt645.exceptions.py
```ruby

```
pydlt645.factory.py
```ruby

```
pydlt645.interfaces.py
```ruby

```
pydlt645.pdu.py
```ruby

```
pydlt645.register_read_message.py
```ruby

```
pydlt645.transaction.py
```ruby

```
pydlt645.utilities.py
```ruby

```
pydlt645.version.py
```ruby

```
pydlt645.client.\__init__.py(empty file) 
pydlt645.client.common.py
```ruby

```
pydlt645.client.sync.py
```ruby

```




## Debug Of pydlt645
## Install Uninstall And Test Of pydlt645



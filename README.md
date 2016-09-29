# 创建Python第三方库——DLT645通讯协议
***
## DLT645 Client written in Python-2.7. Support DLT645-1997/2007
* [IDE](#ide)
* [Get Source Code Of pymodbus](#get-source-code-of-pymodbus)
* [Create Source Code Of pydlt645](#create-source-code-of-pydlt645)
* [Debug pydlt645](#debug-pydlt645)
* [Install、Uninstall And Test pydlt645](#install-uninstall-and-test-pydlt645)
* [FAQ](#faq)

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

## Create Source Code Of pydlt645
**reference pymodbus**

1、Create python project in eclipse IDE.
File|New|Project -> PyDev Project
2、Create the following folders and files.
![p3](http://i.imgur.com/f5PaZSQ.png)

**Code:**

pydlt645.\__init__.py
```ruby
''' Pydlt645: Dlt645 Protocol Implementation '''

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
'''
Pydlt645 Exceptions
--------------------

Custom exceptions to be used in the dlt645 code.
'''

class DLT645Exception(Exception):
    ''' Base dlt645 exception '''

    def __init__(self, string):
        ''' Initialize the exception
        :param string: The message to append to the error
        '''
        self.string = string

    def __str__(self):
        return 'dlt645 Error: %s' % self.string


class DLT645IOException(DLT645Exception):
    ''' Error resulting from data i/o '''

    def __init__(self, string=""):
        ''' Initialize the exception
        :param string: The message to append to the error
        '''
        message = "[Input/Output] %s" % string
        DLT645Exception.__init__(self, message)


class ParameterException(DLT645Exception):
    ''' Error resulting from invalid parameter '''

    def __init__(self, string=""):
        ''' Initialize the exception
        :param string: The message to append to the error
        '''
        message = "[Invalid Parameter] %s" % string
        DLT645Exception.__init__(self, message)


class NoSuchSlaveException(DLT645Exception):
    ''' Error resulting from making a request to a slave that does not exist '''

    def __init__(self, string=""):
        ''' Initialize the exception
        :param string: The message to append to the error
        '''
        message = "[No Such Slave] %s" % string
        DLT645Exception.__init__(self, message)


class NotImplementedException(DLT645Exception):
    ''' Error resulting from not implemented function '''

    def __init__(self, string=""):
        ''' Initialize the exception
        :param string: The message to append to the error
        '''
        message = "[Not Implemented] %s" % string
        DLT645Exception.__init__(self, message)


class ConnectionException(DLT645Exception):
    ''' Error resulting from a bad connection '''

    def __init__(self, string=""):
        ''' Initialize the exception
        :param string: The message to append to the error
        '''
        message = "[Connection] %s" % string
        DLT645Exception.__init__(self, message)

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = ["DLT645Exception", "DLT645IOException","ParameterException", 
           "NotImplementedException","ConnectionException", 
		   "NoSuchSlaveException"]
```
pydlt645.factory.py
```ruby
"""
DLT645 Request/Response Decoder Factories
-------------------------------------------

The following factories make it easy to decode request/response messages.
To add a new request/response pair to be decodeable by the library, simply
add them to the respective function lookup table (order doesn't matter, but
it does help keep things organized).

Regardless of how many functions are added to the lookup, O(1) behavior is
kept as a result of a pre-computed lookup dictionary.
"""

from pydlt645.pdu import IllegalFunctionRequest
from pydlt645.pdu import ExceptionResponse
from pydlt645.pdu import DLT645Exceptions as ecode
from pydlt645.interfaces import IDLT645Decoder
from pydlt645.exceptions import DLT645Exception
from pydlt645.register_read_message import *

#---------------------------------------------------------------------------#
# Logging
#---------------------------------------------------------------------------#
import logging
_logger = logging.getLogger(__name__)

#---------------------------------------------------------------------------#
# Server Decoder
#---------------------------------------------------------------------------#

#---------------------------------------------------------------------------#
# Client Decoder
#---------------------------------------------------------------------------#
class ClientDecoder(IDLT645Decoder):
    ''' Response Message Factory (Client)
    To add more implemented functions, simply add them to the list
    '''
    __function_table = [
            ReadDLT645RegistersResponse,
    ]
    __sub_function_table = [
        
    ]

    def __init__(self):
        ''' Initializes the client lookup tables '''
        functions = set(f.function_code for f in self.__function_table)
        self.__lookup = dict([(f.function_code, f) for f in self.__function_table])
        self.__sub_lookup = dict((f, {}) for f in functions)
        for f in self.__sub_function_table:
            self.__sub_lookup[f.function_code][f.sub_function_code] = f

    def lookupPduClass(self, function_code):
        ''' Use `function_code` to determine the class of the PDU.
        :param function_code: The function code specified in a frame.
        :returns: The class of the PDU that has a matching `function_code`.
        '''
        return self.__lookup.get(function_code, ExceptionResponse)

    def decode(self, message):
        ''' Wrapper to decode a response packet
        :param message: The raw packet to decode
        :return: The decoded DLT645 message or None if error
        '''
        try:
            return self._helper(message)
        except DLT645Exception, er:
            _logger.error("Unable to decode response %s" % er)
        return None

    def _helper(self, data):
        '''
        This factory is used to generate the correct response object
        from a valid response packet. This decodes from a list of the
        currently implemented request types.

        :param data: The response packet to decode
        :returns: The decoded request or an exception response object
        '''
        function_code = 1
        _logger.debug("Factory Response[%d]" % function_code)
        response = self.__lookup.get(function_code, lambda: None)()
        if not response:
            raise DLT645Exception("Unknown response %d" % function_code)
        response.decode(data)

        if hasattr(response, 'sub_function_code'):
            lookup = self.__sub_lookup.get(response.function_code, {})
            subtype = lookup.get(response.sub_function_code, None)
            if subtype: response.__class__ = subtype

        return response

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = ["ClientDecoder"]
```
pydlt645.interfaces.py
```ruby
'''
Pydlt645 Interfaces
---------------------

A collection of base classes that are used throughout the pydlt645 library.
'''
from pydlt645.exceptions import NotImplementedException

#---------------------------------------------------------------------------#
# Generic
#---------------------------------------------------------------------------#
class Singleton(object):
    '''
    Singleton base class
    http://mail.python.org/pipermail/python-list/2007-July/450681.html
    '''
    def __new__(cls, *args, **kwargs):
        ''' Create a new instance '''
        if '_inst' not in vars(cls):
            cls._inst = object.__new__(cls)
        return cls._inst

#---------------------------------------------------------------------------#
# Project Specific
#---------------------------------------------------------------------------#
class IDLT645Decoder(object):
    ''' DLT645 Decoder Base Class

    This interface must be implemented by a dlt645 message
    decoder factory. These factories are responsible for
    abstracting away converting a raw packet into a request / response
    message object.
    '''

    def decode(self, message):
        ''' Wrapper to decode a given packet

        :param message: The raw dlt645 request packet
        :return: The decoded dlt645 message or None if error
        '''
        raise NotImplementedException(
			"Method not implemented by derived class")

    def lookupPduClass(self, function_code):
        ''' Use `function_code` to determine the class of the PDU.

        :param function_code: The function code specified in a frame.
        :returns: The class of the PDU that has a matching `function_code`.
        '''
        raise NotImplementedException(
            "Method not implemented by derived class")

class IDLT645Framer(object):
    '''
    A framer strategy interface. The idea is that we abstract away all the
    detail about how to detect if a current message frame exists, decoding
    it, sending it, etc so that we can plug in a new Framer object (tcp,
    rtu, ascii).
    '''

    def checkFrame(self):
        ''' Check and decode the next frame

        :returns: True if we successful, False otherwise
        '''
        raise NotImplementedException(
            "Method not implemented by derived class")

    def advanceFrame(self):
        ''' Skip over the current framed message
        This allows us to skip over the current message after we have processed
        it or determined that it contains an error. It also has to reset the
        current frame header handle
        '''
        raise NotImplementedException(
            "Method not implemented by derived class")

    def addToFrame(self, message):
        ''' Add the next message to the frame buffer
        This should be used before the decoding while loop to add the received
        data to the buffer handle.
        :param message: The most recent packet
        '''
        raise NotImplementedException(
            "Method not implemented by derived class")

    def isFrameReady(self):
        ''' Check if we should continue decode logic
        This is meant to be used in a while loop in the decoding phase to let
        the decoder know that there is still data in the buffer.
        :returns: True if ready, False otherwise
        '''
        raise NotImplementedException("Method not implemented by derived class")

    def getFrame(self):
        ''' Get the next frame from the buffer
        :returns: The frame data or ''
        '''
        raise NotImplementedException("Method not implemented by derived class")

    def populateResult(self, result):
        ''' Populates the dlt645 result with current frame header

        We basically copy the data back over from the current header
        to the result header. This may not be needed for serial messages.
        :param result: The response packet
        '''
        raise NotImplementedException("Method not implemented by derived class")

    def processIncomingPacket(self, data, callback):
        ''' The new packet processing pattern

        This takes in a new request packet, adds it to the current
        packet stream, and performs framing on it. That is, checks
        for complete messages, and once found, will process all that
        exist.  This handles the case when we read N + 1 or 1 / N
        messages at a time instead of 1.

        The processed and decoded messages are pushed to the callback
        function to process and send.

        :param data: The new packet data
        :param callback: The function to send results to
        '''
        raise NotImplementedException("Method not implemented by derived class")

    def buildPacket(self, message):
        ''' Creates a ready to send dlt645 packet

        The raw packet is built off of a fully populated dlt645
        request / response message.
        :param message: The request/response to send
        :returns: The built packet
        '''
        raise NotImplementedException("Method not implemented by derived class")

class IDLT645SlaveContext(object):
    '''
    Interface for a dlt645 slave data context

    Derived classes must implemented the following methods:
            reset(self)
            validate(self, fx, address, count=1)
            getValues(self, fx, address, count=1)
            setValues(self, fx, address, values)
    '''
    __fx_mapper = {2: 'd', 4: 'i'}
    __fx_mapper.update([(i, 'h') for i in [3, 6, 16, 22, 23]])
    __fx_mapper.update([(i, 'c') for i in [1, 5, 15]])

    def decode(self, fx):
        ''' Converts the function code to the datastore to
        :param fx: The function we are working with
        :returns: one of [d(iscretes),i(inputs),h(oliding),c(oils)
        '''
        return self.__fx_mapper[fx]

    def reset(self):
        ''' Resets all the datastores to their default values '''
        raise NotImplementedException("Context Reset")

    def validate(self, fx, address, count=1):
        ''' Validates the request to make sure it is in range
        :param fx: The function we are working with
        :param address: The starting address
        :param count: The number of values to test
        :returns: True if the request in within range, False otherwise
        '''
        raise NotImplementedException("validate context values")

    def getValues(self, fx, address, count=1):
        ''' Validates the request to make sure it is in range
        :param fx: The function we are working with
        :param address: The starting address
        :param count: The number of values to retrieve
        :returns: The requested values from a:a+c
        '''
        raise NotImplementedException("get context values")

    def setValues(self, fx, address, values):
        ''' Sets the datastore with the supplied values
        :param fx: The function we are working with
        :param address: The starting address
        :param values: The new values to be set
        '''
        raise NotImplementedException("set context values")

class IPayloadBuilder(object):
    '''
    This is an interface to a class that can build a payload
    for a modbus register write command. It should abstract
    the codec for encoding data to the required format
    (bcd, binary, char, etc).
    '''

    def build(self):
        ''' Return the payload buffer as a list

        This list is two bytes per element and can
        thus be treated as a list of registers.

        :returns: The payload buffer as a list
        '''
        raise NotImplementedException("set context values")

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = ['Singleton', 'IDLT645Decoder', 'IDLT645Framer', 
    	   'IDLT645SlaveContext','IPayloadBuilder']
```
pydlt645.pdu.py
```ruby
'''
Contains base classes for dlt645 request/response/error packets
'''
from pydlt645.interfaces import Singleton
from pydlt645.exceptions import NotImplementedException
from pydlt645.constants import Defaults
from pydlt645.utilities import rtuFrameSize

#---------------------------------------------------------------------------#
# Logging
#---------------------------------------------------------------------------#
import logging
_logger = logging.getLogger(__name__)

#---------------------------------------------------------------------------#
# Base PDU's
#---------------------------------------------------------------------------#
class DLT645PDU(object):
    '''
    Base class for all DLT645 mesages

    .. attribute:: transaction_id

       This value is used to uniquely identify a request
       response pair.  It can be implemented as a simple counter

    .. attribute:: protocol_id

       This is a constant set at 0 to indicate DLT645.  It is
       put here for ease of expansion.

    .. attribute:: check

       This is used for CS in the serial dlt645 protocols

    .. attribute:: skip_encode

       This is used when the message payload has already been encoded.
       Generally this will occur when the PayloadBuilder is being used
       to create a complicated message. By setting this to True, the
       request will pass the currently encoded message through instead
       of encoding it again.
    '''

    def __init__(self, **kwargs):
        ''' Initializes the base data for a dlt645 request '''
        self.transaction_id = kwargs.get('transaction', Defaults.TransactionId)
        self.protocol_id = kwargs.get('protocol', Defaults.ProtocolId)
        self.skip_encode = kwargs.get('skip_encode', False)
        self.check = 0x00

    def encode(self):
        ''' Encodes the message

        :raises: A not implemented exception
        '''
        raise NotImplementedException()

    def decode(self, data):
        ''' Decodes data part of the message.

        :param data: is a string object
        :raises: A not implemented exception
        '''
        raise NotImplementedException()

    @classmethod
    def calculateRtuFrameSize(cls, buffer):
        ''' Calculates the size of a PDU.

        :param buffer: A buffer containing the data that have been received.
        :returns: The number of bytes in the PDU.
        '''
        if hasattr(cls, '_rtu_frame_size'):
            return cls._rtu_frame_size
        elif hasattr(cls, '_rtu_byte_count_pos'):
            return rtuFrameSize(buffer, cls._rtu_byte_count_pos)
        else: raise NotImplementedException(
            "Cannot determine RTU frame size for %s" % cls.__name__)


class DLT645Request(DLT645PDU):
    ''' Base class for a dlt645 request PDU '''

    def __init__(self, **kwargs):
        ''' Proxy to the lower level initializer '''
        DLT645PDU.__init__(self, **kwargs)

    def doException(self, exception):
        ''' Builds an error response based on the function

        :param exception: The exception to return
        :raises: An exception response
        '''
        _logger.error("Exception Response F(%d) E(%d)" %
                (self.function_code, exception))
        return ExceptionResponse(self.function_code, exception)


class DLT645Response(DLT645PDU):
    ''' Base class for a dlt645 response PDU

    .. attribute:: should_respond

       A flag that indicates if this response returns a result back
       to the client issuing the request

    .. attribute:: _rtu_frame_size

       Indicates the size of the dlt645 rtu response used for
       calculating how much to read.
    '''

    should_respond = True

    def __init__(self, **kwargs):
        ''' Proxy to the lower level initializer '''
        DLT645PDU.__init__(self, **kwargs)

#---------------------------------------------------------------------------#
# Exception PDU's
#---------------------------------------------------------------------------#
class DLT645Exceptions(Singleton):
    '''
    An enumeration of the valid dlt645 exceptions
    '''
    IllegalFunction         = 0x01
    IllegalAddress          = 0x02
    IllegalValue            = 0x03
    SlaveFailure            = 0x04
    Acknowledge             = 0x05
    SlaveBusy               = 0x06
    MemoryParityError       = 0x08
    GatewayPathUnavailable  = 0x0A
    GatewayNoResponse       = 0x0B

    @classmethod
    def decode(cls, code):
        ''' Given an error code, translate it to a
        string error name. 
        
        :param code: The code number to translate
        '''
        values = dict((v, k) for k, v in cls.__dict__.iteritems()
            if not k.startswith('__') and not callable(v))
        return values.get(code, None)

class ExceptionResponse(DLT645Response):
    ''' Base class for a dlt645 exception PDU '''
    ExceptionOffset = 0x80
    _rtu_frame_size = 5

    def __init__(self, function_code, exception_code=None, **kwargs):
        ''' Initializes the dlt645 exception response

        :param function_code: The function to build an exception response for
        :param exception_code: The specific dlt645 exception to return
        '''
        DLT645Response.__init__(self, **kwargs)
        self.original_code = function_code
        self.function_code = function_code | self.ExceptionOffset
        self.exception_code = exception_code

    def encode(self):
        ''' Encodes a dlt645 exception response

        :returns: The encoded exception packet
        '''
        return chr(self.exception_code)

    def decode(self, data):
        ''' Decodes a dlt645 exception response

        :param data: The packet data to decode
        '''
        self.exception_code = ord(data[0])

    def __str__(self):
        ''' Builds a representation of an exception response

        :returns: The string representation of an exception response
        '''
        message = DLT645Exceptions.decode(self.exception_code)
        parameters = (self.function_code, self.original_code, message)
        return "Exception Response(%d, %d, %s)" % parameters

class IllegalFunctionRequest(DLT645Request):
    '''
    Defines the DLT645 slave exception type 'Illegal Function'
    This exception code is returned if the slave::

        - does not implement the function code **or**
        - is not in a state that allows it to process the function
    '''
    ErrorCode = 1

    def __init__(self, function_code, **kwargs):
        ''' Initializes a IllegalFunctionRequest

        :param function_code: The function we are erroring on
        '''
        DLT645Request.__init__(self, **kwargs)
        self.function_code = function_code

    def decode(self, data):
        ''' This is here so this failure will run correctly

        :param data: Not used
        '''
        pass

    def execute(self, context):
        ''' Builds an illegal function request error response

        :param context: The current context for the message
        :returns: The error response packet
        '''
        return ExceptionResponse(self.function_code, self.ErrorCode)

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = ['DLT645Request', 'DLT645Response', 'DLT645Exceptions',
           'ExceptionResponse', 'IllegalFunctionRequest']
```
pydlt645.register_read_message.py
```ruby
#-*- coding: UTF-8 -*- 
'''
Register Reading Request/Response
---------------------------------
'''
import struct
from pydlt645.pdu import DLT645Request
from pydlt645.pdu import DLT645Response
from pydlt645.pdu import DLT645Exceptions as merror
import pydlt645.transaction as paraset

class ReadRegistersRequestBase(DLT645Request):
    ''' Base class for reading a dlt645 register '''

    def __init__(self, version, address, dataid, count, **kwargs):
        ''' Initializes a new instance
        '''
        DLT645Request.__init__(self, **kwargs)
        paraset.dlt645VERSION = self.version = version
        paraset.dlt645ADDRESS = self.address = address
        paraset.dlt645DATAID = self.dataid = dataid
        
        self.count = count

    def encode(self):
        ''' Encodes the request packet
        address(A0~A5)+帧起始符(0x68)+控制码C+数据域长度L+数据项标识(dataid)
        :return: The encoded packet
        '''
        #address(A0~A5)
        temp = ''
        for i in range(0,6):
            temp +=struct.pack('>B', (self.address >> (i * 8)) & 0xFF)
        
        #帧起始符(0x68)
        temp +=struct.pack('>B', 0x68)
        
        #控制码C(读数据:dlt645-1997:0x01,dlt645-2007:0x11)
        #数据长度L(dlt645-1997:2字节,dlt645-2007:4字节)
        if self.version == '1997':
            temp +=struct.pack('>B', 0x01)
            temp +=struct.pack('>B', 2)
            #数据项标识(dataid)
            temp +=struct.pack('>BB', (self.dataid & 0xFF) + 0x33, \
                                      ((self.dataid >> 8) & 0xFF) + 0x33)
        else:
            temp +=struct.pack('>B', 0x11)
            temp +=struct.pack('>B', 4)
            #数据项标识(dataid)
            for i in range(0,4):
                temp +=struct.pack('>B', ((self.dataid >> (i * 8)) & 0xFF) + 0x33)
        return temp

    def decode(self, data):
        ''' Decode a register request packet
        :param data: The request to decode
        '''
        
    def __str__(self):
        ''' Returns a string representation of the instance

        :returns: A string representation of the instance
        '''
        return "ReadRegisterRequest (%x,%d)" % (self.address, self.count)


class ReadRegistersResponseBase(DLT645Response):
    ''' Base class for responsing to a dlt645 register read '''

    def __init__(self, values, **kwargs):
        ''' Initializes a new instance

        :param values: The values to write to
        '''
        DLT645Response.__init__(self, **kwargs)
        self.registers = values or []

    def encode(self):
        ''' Encodes the response packet

        :returns: The encoded packet
        '''

    def decode(self, data):
        ''' Decode a register response packet

        :param data: The request to decode
        '''
        self.registers = []
        for i in range(0, len(data)):
            self.registers.append(struct.unpack('>B', data[i])[0] - 0x33)

    def getRegister(self, index):
        ''' Get the requested register

        :param index: The indexed register to retrieve
        :returns: The request register
        '''
        return self.registers[index]

    def __str__(self):
        ''' Returns a string representation of the instance

        :returns: A string representation of the instance
        '''
        return "ReadRegisterResponse (%d)" % len(self.registers)

class ReadDLT645RegistersRequest(ReadRegistersRequestBase):
  
    function_code = 1

    def __init__(self, version=None, address=None, dataid=None, count=None, **kwargs):
        ''' Initializes a new instance of the request
        '''
        ReadRegistersRequestBase.__init__(self, version, address, dataid, count, **kwargs)
    
    def execute(self, context):
        ''' Run a read holding request against a datastore

        :param context: The datastore to request from
        :returns: An initialized response, exception message otherwise
        '''
        return ReadDLT645RegistersResponse(values)

class ReadDLT645RegistersResponse(ReadRegistersResponseBase):
    '''
    This function code is used to read the contents of a contiguous block
    of holding registers in a remote device. The Request PDU specifies the
    starting register address and the number of registers. In the PDU
    Registers are addressed starting at zero. Therefore registers numbered
    1-16 are addressed as 0-15.
    '''
    function_code = 1

    def __init__(self, values=None, **kwargs):
        ''' Initializes a new response instance

        :param values: The resulting register values
        '''
        ReadRegistersResponseBase.__init__(self, values, **kwargs)

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = [
    "ReadDLT645RegistersRequest", "ReadDLT645RegistersResponse",
]
```
pydlt645.transaction.py
```ruby
#-*- coding: UTF-8 -*- 
''' Collection of transaction based abstractions '''
import sys
import struct
import socket
# from binascii import b2a_hex, a2b_hex

from pydlt645.exceptions import DLT645IOException
from pydlt645.constants  import Defaults
from pydlt645.interfaces import IDLT645Framer
from pydlt645.utilities  import checkCS, computeCS

#---------------------------------------------------------------------------#
# Logging
#---------------------------------------------------------------------------#
import logging
_logger = logging.getLogger(__name__)

dlt645VERSION = ''
dlt645ADDRESS = 0
dlt645DATAID  = 0

#---------------------------------------------------------------------------#
# The Global Transaction Manager
#---------------------------------------------------------------------------#
class DLT645TransactionManager(object):
    ''' Impelements a transaction for a manager
    The transaction protocol can be represented by the following pseudo code::

        count = 0
        do
          result = send(message)
          if (timeout or result == bad)
             count++
          else break
        while (count < 3)

    This module helps to abstract this away from the framer and protocol.
    '''

    def __init__(self, client, **kwargs):
        ''' Initializes an instance of the DLT645TransactionManager

        :param client: The client socket wrapper
        :param retry_on_empty: Should the client retry on empty
        :param retries: The number of retries to allow
        '''
        self.tid = Defaults.TransactionId
        self.client = client
        self.retry_on_empty = kwargs.get('retry_on_empty', Defaults.RetryOnEmpty)
        self.retries = kwargs.get('retries', Defaults.Retries)

    def execute(self, request):
        ''' Starts the producer to send the next request to
        consumer.write(Frame(request))
        '''
        retries = self.retries
        request.transaction_id = self.getNextTID()
        _logger.debug("Running transaction %d" % request.transaction_id)

        while retries > 0:
            try:
                self.client.connect()
                self.client._send(self.client.framer.buildPacket(request))
                # I need to fix this to read the header and the result size,
                # as this may not read the full result set, but right now
                # it should be fine...
                result = self.client._recv(1024)
                if not result and self.retry_on_empty:
                    retries -= 1
                    continue
                if _logger.isEnabledFor(logging.DEBUG):
                    _logger.debug("recv: " + " ".join([hex(ord(x)) for x in result]))
                self.client.framer.processIncomingPacket(result, self.addTransaction)
                break;
            except socket.error, msg:
                self.client.close()
                _logger.debug("Transaction failed. (%s) " % msg)
                retries -= 1
        return self.getTransaction(request.transaction_id)

    def addTransaction(self, request, tid=None):
        ''' Adds a transaction to the handler

        This holds the requets in case it needs to be resent.
        After being sent, the request is removed.

        :param request: The request to hold on to
        :param tid: The overloaded transaction id to use
        '''
        raise NotImplementedException("addTransaction")

    def getTransaction(self, tid):
        ''' Returns a transaction matching the referenced tid

        If the transaction does not exist, None is returned

        :param tid: The transaction to retrieve
        '''
        raise NotImplementedException("getTransaction")

    def delTransaction(self, tid):
        ''' Removes a transaction matching the referenced tid

        :param tid: The transaction to remove
        '''
        raise NotImplementedException("delTransaction")

    def getNextTID(self):
        ''' Retrieve the next unique transaction identifier

        This handles incrementing the identifier after
        retrieval

        :returns: The next unique transaction identifier
        '''
        self.tid = (self.tid + 1) & 0xffff
        return self.tid

    def reset(self):
        ''' Resets the transaction identifier '''
        self.tid = Defaults.TransactionId
        self.transactions = type(self.transactions)()

class FifoTransactionManager(DLT645TransactionManager):
    ''' Impelements a transaction for a manager where the
    results are returned in a FIFO manner.
    '''

    def __init__(self, client, **kwargs):
        ''' Initializes an instance of the DLT645TransactionManager

        :param client: The client socket wrapper
        '''
        super(FifoTransactionManager, self).__init__(client, **kwargs)
        self.transactions = []

    def __iter__(self):
        ''' Iterater over the current managed transactions

        :returns: An iterator of the managed transactions
        '''
        return iter(self.transactions)

    def addTransaction(self, request, tid=None):
        ''' Adds a transaction to the handler

        This holds the requets in case it needs to be resent.
        After being sent, the request is removed.

        :param request: The request to hold on to
        :param tid: The overloaded transaction id to use
        '''
        tid = tid if tid != None else request.transaction_id
        _logger.debug("adding transaction %d" % tid)
        self.transactions.append(request)

    def getTransaction(self, tid):
        ''' Returns a transaction matching the referenced tid

        If the transaction does not exist, None is returned

        :param tid: The transaction to retrieve
        '''
        _logger.debug("getting transaction %s" % str(tid))
        return self.transactions.pop(0) if self.transactions else None

    def delTransaction(self, tid):
        ''' Removes a transaction matching the referenced tid

        :param tid: The transaction to remove
        '''
        _logger.debug("deleting transaction %d" % tid)
        if self.transactions: self.transactions.pop(0)

#---------------------------------------------------------------------------#
# DLT645 Message
#---------------------------------------------------------------------------#
class DLT645Framer(IDLT645Framer):
    '''
    DLT645 Frame Controller::

        [ Start ][Address ][ Start ][ C ][ L ][ DataId + Data][ CS ][ End ]
          68        A0~A5     68                                      16

    This framer is used for serial transmission.
    '''

    def __init__(self, decoder):
        ''' Initializes a new instance of the framer

        :param decoder: The decoder implementation to use
        '''
        self.__buffer = ''
        self.__header = {'cs':0, 'len':0}
        self.__hsize  = 0x00
        self.__start  = 0x68
        self.__end    = 0x16
        self.decoder  = decoder

    #-----------------------------------------------------------------------#
    # Private Helper Functions
    #-----------------------------------------------------------------------#
    def checkFrame(self):
        ''' Check and decode the next frame

        :returns: True if we successful, False otherwise
        '''
        version = dlt645VERSION
        address = dlt645ADDRESS
        dataid = dlt645DATAID
        start = -1
        for i in self.__buffer:
            if ord(i) == self.__start:
                start = self.__buffer.index(i)
                break
        if start < 0: return False
        
        self.__buffer = self.__buffer[start:]
        start = 0
        self.__hsize +=1
        
        end = -1
        for i in self.__buffer:
            if ord(i) == self.__end:
                end = self.__buffer.index(i)
                break
        if end < 0: return False
        
        self.__header['len'] = end
        self.__header['cs'] = struct.unpack('>B', self.__buffer[end - 1])[0]
        buftemp = self.__buffer[start:end-1]
        if not checkCS(buftemp, self.__header['cs']):
            return False
            
        #address(A0~A5)
        addrtemp = 0
        for i in range(0, 6):
            addrtemp +=(struct.unpack('>B', self.__buffer[i + self.__hsize])[0] & 0xFF ) << (i * 8) 
        if addrtemp != address: return False
        self.__hsize +=6
        
        #帧起始符
        if struct.unpack('>B', self.__buffer[self.__hsize])[0] != self.__start: return False
        self.__hsize +=1
        
        #控制码C(读数据回应:dlt645-1997:0x81,dlt645-2007:0x91)
        ctltemp = struct.unpack('>B', self.__buffer[self.__hsize])[0]
        self.__hsize +=1
        
        #数据长度L
        self.__hsize +=1
        
        if version == '1997':
            if ctltemp != 0x81 : return False
            #数据项标识(dataid)
            dataidtemp = (struct.unpack('>B', self.__buffer[self.__hsize])[0] - 0x33) + \
                         ((struct.unpack('>B', self.__buffer[self.__hsize + 1])[0] - 0x33) << 8)
            if dataidtemp != dataid : return False
            self.__hsize +=2
        else:
            if ctltemp != 0x91 : return False
            #数据项标识(dataid)
            dataidtemp = (struct.unpack('>B', self.__buffer[self.__hsize])[0] - 0x33) + \
                         ((struct.unpack('>B', self.__buffer[self.__hsize + 1])[0] - 0x33) << 8) + \
                         ((struct.unpack('>B', self.__buffer[self.__hsize + 2])[0] - 0x33) << 16) + \
                         ((struct.unpack('>B', self.__buffer[self.__hsize + 3])[0] - 0x33) << 24)
            if dataidtemp != dataid : return False
            self.__hsize +=4
        
        return True

    def advanceFrame(self):
        ''' Skip over the current framed message
        This allows us to skip over the current message after we have processed
        it or determined that it contains an error. It also has to reset the
        current frame header handle
        '''
        self.__buffer = ''
        self.__hsize  = 0x00
        self.__header = {'cs':0, 'len':0}

    def isFrameReady(self):
        ''' Check if we should continue decode logic
        This is meant to be used in a while loop in the decoding phase to let
        the decoder know that there is still data in the buffer.

        :returns: True if ready, False otherwise
        '''
        return len(self.__buffer) > 1

    def addToFrame(self, message):
        ''' Add the next message to the frame buffer
        This should be used before the decoding while loop to add the received
        data to the buffer handle.

        :param message: The most recent packet
        '''
        self.__buffer += message

    def getFrame(self):
        ''' Get the next frame from the buffer

        :returns: The frame data or ''
        '''
        start  = self.__hsize
        end    = self.__header['len'] - 1
        return self.__buffer[start:end]

    def populateResult(self, result):
        ''' Populates the dlt645 result header

        The serial packets do not have any header information
        that is copied.

        :param result: The response packet
        '''

    #-----------------------------------------------------------------------#
    # Public Member Functions
    #-----------------------------------------------------------------------#
    def processIncomingPacket(self, data, callback):
        ''' The new packet processing pattern

        This takes in a new request packet, adds it to the current
        packet stream, and performs framing on it. That is, checks
        for complete messages, and once found, will process all that
        exist.  This handles the case when we read N + 1 or 1 / N
        messages at a time instead of 1.

        The processed and decoded messages are pushed to the callback
        function to process and send.

        :param data: The new packet data
        :param callback: The function to send results to
        '''
        self.addToFrame(data)
        while self.isFrameReady():
            if self.checkFrame():
                result = self.decoder.decode(self.getFrame())
                if result is None:
                    raise DLT645IOException("Unable to decode response")
                self.populateResult(result)
                self.advanceFrame()
                callback(result)  # defer this
            else: break

    def buildPacket(self, message):
        ''' Creates a ready to send dlt645 packet
        Built off of a  dlt645 request/response

        :param message: The request/response to send
        :return: The encoded packet
        '''
        start = struct.pack('>BBBB', 0xFE, 0xFE, 0xFE, 0xFE)
        framestart = struct.pack('>B', self.__start)
        encoded  = message.encode()
        packet = start + framestart + encoded
        
        checksum = computeCS(framestart + encoded)
        packet += struct.pack('>B', checksum)
        
        packet += struct.pack('>B', self.__end)
        
        return packet
    
#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = ["FifoTransactionManager", "DLT645Framer"]
```
pydlt645.utilities.py
```ruby
'''
DLT645 Utilities
-----------------

A collection of utilities for packing data, unpacking
data computing checksums, and decode checksums.
'''
import struct

#---------------------------------------------------------------------------#
# Helpers
#---------------------------------------------------------------------------#
def default(value):
    '''
    Given a python object, return the default value of that object.

    :param value: The value to get the default of
    :returns: The default value
    '''
    return type(value)()


def dict_property(store, index):
    ''' Helper to create class properties from a dictionary. Basically this 
       allows you to remove a lot of possible boilerplate code.

    :param store: The store store to pull from
    :param index: The index into the store to close over
    :returns: An initialized property set
    '''
    if hasattr(store, '__call__'):
        getter = lambda self: store(self)[index]
        setter = lambda self, value: store(self).__setitem__(index, value)
    elif isinstance(store, str):
        getter = lambda self: self.__getattribute__(store)[index]
        setter = lambda self, value: self.__getattribute__(store).__setitem__(
            index, value)
    else:
        getter = lambda self: store[index]
        setter = lambda self, value: store.__setitem__(index, value)

    return property(getter, setter)


#---------------------------------------------------------------------------#
# Bit packing functions
#---------------------------------------------------------------------------#
def pack_bitstring(bits):
    ''' Creates a string out of an array of bits

    :param bits: A bit array

    example::

        bits   = [False, True, False, True]
        result = pack_bitstring(bits)
    '''
    ret = ''
    i = packed = 0
    for bit in bits:
        if bit: packed += 128
        i += 1
        if i == 8:
            ret += chr(packed)
            i = packed = 0
        else: packed >>= 1
    if i > 0 and i < 8:
        packed >>= (7 - i)
        ret += chr(packed)
    return ret

def unpack_bitstring(string):
    ''' Creates bit array out of a string

    :param string: The modbus data packet to decode

    example::

        bytes  = 'bytes to decode'
        result = unpack_bitstring(bytes)
    '''
    byte_count = len(string)
    bits = []
    for byte in range(byte_count):
        value = ord(string[byte])
        for _ in range(8):
            bits.append((value & 1) == 1)
            value >>= 1
    return bits

#---------------------------------------------------------------------------#
# Error Detection Functions
#---------------------------------------------------------------------------#
def __generate_crc16_table():
    ''' Generates a crc16 lookup table
    .. note:: This will only be generated once
    '''
    result = []
    for byte in range(256):
        crc = 0x0000
        for _ in range(8):
            if (byte ^ crc) & 0x0001:
                crc = (crc >> 1) ^ 0xa001
            else: crc >>= 1
            byte >>= 1
        result.append(crc)
    return result

__crc16_table = __generate_crc16_table()


def computeCRC(data):
    ''' Computes a crc16 on the passed in string. For modbus,
    this is only used on the binary serial protocols (in this
    case RTU).

    The difference between modbus's crc16 and a normal crc16
    is that modbus starts the crc value out at 0xffff.

    :param data: The data to create a crc16 of
    :returns: The calculated CRC
    '''
    crc = 0xffff
    for a in data:
        idx = __crc16_table[(crc ^ ord(a)) & 0xff];
        crc = ((crc >> 8) & 0xff) ^ idx
    swapped = ((crc << 8) & 0xff00) | ((crc >> 8) & 0x00ff)
    return swapped


def checkCRC(data, check):
    ''' Checks if the data matches the passed in CRC

    :param data: The data to create a crc16 of
    :param check: The CRC to validate
    :returns: True if matched, False otherwise
    '''
    return computeCRC(data) == check


def computeLRC(data):
    ''' Used to compute the longitudinal redundancy check against a string. 
    This is only used on the serial ASCII modbus protocol. A full description
    of this implementation can be found in appendex B of the serial line modbus 
    description.

    :param data: The data to apply a lrc to
    :returns: The calculated LRC

    '''
    lrc = sum(ord(a) for a in data) & 0xff
    lrc = (lrc ^ 0xff) + 1
    return lrc & 0xff

def checkLRC(data, check):
    ''' Checks if the passed in data matches the LRC

    :param data: The data to calculate
    :param check: The LRC to validate
    :returns: True if matched, False otherwise
    '''
    return computeLRC(data) == check


def rtuFrameSize(buffer, byte_count_pos):
    ''' Calculates the size of the frame based on the byte count.

    :param buffer: The buffer containing the frame.
    :param byte_count_pos: The index of the byte count in the buffer.
    :returns: The size of the frame.

    The structure of frames with a byte count field is always the
    same:

    - first, there are some header fields
    - then the byte count field
    - then as many data bytes as indicated by the byte count,
    - finally the CRC (two bytes).

    To calculate the frame size, it is therefore sufficient to extract
    the contents of the byte count field, add the position of this
    field, and finally increment the sum by three (one byte for the
    byte count field, two for the CRC).
    '''
    return struct.unpack('>B', buffer[byte_count_pos])[0] + byte_count_pos + 3

def computeCS(data):
    cssum = 0
    for a in data:
        cssum +=ord(a)
        cssum &= 0xFF
    return cssum

def checkCS(data, check):
    return computeCS(data) == check

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = ['pack_bitstring', 'unpack_bitstring', 'default','computeCRC', 'checkCRC', 
           'computeLRC', 'checkLRC', 'rtuFrameSize','computeCS','checkCS']
```
pydlt645.version.py
```ruby
'''
Handle the version information here; you should only have to
change the version tuple.

Since we are using twisted's version class, we can also query
the svn version as well using the local .entries file.
'''

class Version(object):

    def __init__(self, package, major, minor, micro):
        '''
        :param package: Name of the package that this is a version of.
        :param major: The major version number.
        :param minor: The minor version number.
        :param micro: The micro version number.
        '''
        self.package = package
        self.major = major
        self.minor = minor
        self.micro = micro

    def short(self):
        ''' Return a string in canonical short version format
        <major>.<minor>.<micro>
        '''
        return '%d.%d.%d' % (self.major, self.minor, self.micro)

    def __str__(self):
        ''' Returns a string representation of the object

        :returns: A string representation of this object
        '''
        return '[%s, version %s]' % (self.package, self.short())

version = Version('pydlt645', 1, 0, 0)
version.__name__ = 'pydlt645'  # fix epydoc error

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = ["version"]
```
pydlt645.client.\__init__.py(empty file) 
pydlt645.client.common.py
```ruby
'''
DLT645-1997/2007 Common
----------------------------------

This is a common client mixin that can be used by
clients to simplify the interface.
'''
from pydlt645.register_read_message import *

class DLT645ClientMixin(object):
   
    def read_dlt645_registers(self, version, address, dataid, count=4, **kwargs):
        '''
        :param version:The version of dlt645(1997/2007)
        :param address: The slave address to read from
        :param dataid: dlt645-1997:0x9010  dlt645-2007:0x10000
        :param count: The number of registers to read
        
        :returns: A deferred response handle
        '''
        request = ReadDLT645RegistersRequest(version, address, dataid, count, **kwargs)
        return self.execute(request) 

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = [ 'DLT645ClientMixin' ]
```
pydlt645.client.sync.py
```ruby
import socket
import serial

from pydlt645.constants import Defaults
from pydlt645.factory import ClientDecoder
from pydlt645.exceptions import NotImplementedException, ParameterException
from pydlt645.exceptions import ConnectionException
from pydlt645.transaction import FifoTransactionManager
from pydlt645.transaction import DLT645Framer
from pydlt645.client.common import DLT645ClientMixin

#---------------------------------------------------------------------------#
# Logging
#---------------------------------------------------------------------------#
import logging
_logger = logging.getLogger(__name__)

#---------------------------------------------------------------------------#
# The Synchronous Clients
#---------------------------------------------------------------------------#
class BaseDLT645Client(DLT645ClientMixin):
    '''
    Inteface for a dlt645 synchronous client. Defined here are all the
    methods for performing the related request methods.  Derived classes
    simply need to implement the transport methods and set the correct
    framer.
    '''

    def __init__(self, framer, **kwargs):
        ''' Initialize a client instance

        :param framer: The dlt645 framer implementation to use
        '''
        self.framer = framer
        self.transaction = FifoTransactionManager(self, **kwargs)

    #-----------------------------------------------------------------------#
    # Client interface
    #-----------------------------------------------------------------------#
    def connect(self):
        ''' Connect to the dlt645 remote host

        :returns: True if connection succeeded, False otherwise
        '''
        raise NotImplementedException("Method not implemented by derived class")

    def close(self):
        ''' Closes the underlying socket connection
        '''
        pass

    def _send(self, request):
        ''' Sends data on the underlying socket

        :param request: The encoded request to send
        :return: The number of bytes written
        '''
        raise NotImplementedException("Method not implemented by derived class")

    def _recv(self, size):
        ''' Reads data from the underlying descriptor

        :param size: The number of bytes to read
        :return: The bytes read
        '''
        raise NotImplementedException("Method not implemented by derived class")

    #-----------------------------------------------------------------------#
    # DLT645 client methods
    #-----------------------------------------------------------------------#
    def execute(self, request=None):
        '''
        :param request: The request to process
        :returns: The result of the request execution
        '''
        if not self.connect():
            raise ConnectionException("Failed to connect[%s]" % (self.__str__()))
        return self.transaction.execute(request)

    #-----------------------------------------------------------------------#
    # The magic methods
    #-----------------------------------------------------------------------#
    def __enter__(self):
        ''' Implement the client with enter block

        :returns: The current instance of the client
        '''
        if not self.connect():
            raise ConnectionException("Failed to connect[%s]" % (self.__str__()))
        return self

    def __exit__(self, klass, value, traceback):
        ''' Implement the client with exit block '''
        self.close()

    def __str__(self):
        ''' Builds a string representation of the connection

        :returns: The string representation
        '''
        return "Null Transport"

#---------------------------------------------------------------------------#
# DLT645 Client Transport Implementation
#---------------------------------------------------------------------------#
class DLT645SerialClient(BaseDLT645Client):
    ''' Implementation of a dlt645 serial client '''

    def __init__(self, method='dlt645', **kwargs):
        ''' Initialize a serial client instance

        The methods to connect is::

          - dlt645

        :param method: The method to use for connection
        :param port: The serial port to attach to
        :param stopbits: The number of stop bits to use
        :param bytesize: The bytesize of the serial messages
        :param parity: Which kind of parity to use
        :param baudrate: The baud rate to use for the serial device
        :param timeout: The timeout between serial requests (default 3s)
        '''
        self.method   = method
        self.socket   = None
        BaseDLT645Client.__init__(self, self.__implementation(method), **kwargs)

        self.port     = kwargs.get('port', 0)
        self.stopbits = kwargs.get('stopbits', Defaults.Stopbits)
        self.bytesize = kwargs.get('bytesize', Defaults.Bytesize)
        self.parity   = kwargs.get('parity',   Defaults.Parity)
        self.baudrate = kwargs.get('baudrate', Defaults.Baudrate)
        self.timeout  = kwargs.get('timeout',  Defaults.Timeout)

    @staticmethod
    def __implementation(method):
        ''' Returns the requested framer

        :method: The serial framer to instantiate
        :returns: The requested serial framer
        '''
        method = method.lower()
        if method == 'dlt645':    return DLT645Framer(ClientDecoder())
        raise ParameterException("Invalid framer method requested")

    def connect(self):
        ''' Connect to the dlt645 serial server

        :returns: True if connection succeeded, False otherwise
        '''
        if self.socket: return True
        try:
            self.socket = serial.Serial(port=self.port, timeout=self.timeout,
                bytesize=self.bytesize, stopbits=self.stopbits,
                baudrate=self.baudrate, parity=self.parity)
        except serial.SerialException, msg:
            _logger.error(msg)
            self.close()
        return self.socket != None

    def close(self):
        ''' Closes the underlying socket connection
        '''
        if self.socket:
            self.socket.close()
        self.socket = None

    def _send(self, request):
        ''' Sends data on the underlying socket

        :param request: The encoded request to send
        :return: The number of bytes written
        '''
        if not self.socket:
            raise ConnectionException(self.__str__())
        if request:
            return self.socket.write(request)
        return 0

    def _recv(self, size):
        ''' Reads data from the underlying descriptor

        :param size: The number of bytes to read
        :return: The bytes read
        '''
        if not self.socket:
            raise ConnectionException(self.__str__())
        return self.socket.read(size)

    def __str__(self):
        ''' Builds a string representation of the connection

        :returns: The string representation
        '''
        return "%s baud[%s]" % (self.method, self.baudrate)

#---------------------------------------------------------------------------#
# Exported symbols
#---------------------------------------------------------------------------#
__all__ = [
    "DLT645SerialClient"
]
```

## Debug pydlt645
1、Create test.py file in project.
2、Edit test.py as following.
```ruby
#!/usr/bin/env python
#-*- coding: UTF-8 -*-

import serial
from pydlt645.client.sync import DLT645SerialClient as SerialClient

client = SerialClient(method = "dlt645", port = "/dev/ttyUSB0", stopbits = 1, bytesize = 8, parity = 'E', baudrate= 1200, timeout = 3)

result = client.read_dlt645_registers(version = '1997', address = 0x000003410201, dataid=0x9010)
if result:
	print result.registers
    j  = 1
    DataValue = 0
    for i in range(0, DataLen):
        DataValue += BCD2BIN(result.registers[i]) * j
        j *=100
    print "DataValue = %.2f" % (DataValue / 100.0)
client.close()
```
3、select Debug and Debug with breakpoints as following.
![p4](http://i.imgur.com/0jjASxr.png)
We can find debug message from the Console.

## Install Uninstall And Test pydlt645
### Modify setup.py、setup_commands.py、ez_setup.py in pymodbus about some name and version infomation
### Install pydlt645 in Python-2.7 base on ubuntu16.04
**~$cd pydlt645-master**
**~/pydlt645-master$sudo python setup.py install**

result1:
~$pip list
...
pydlt645(1.0.0)
...
result2:
~$cd /usr/local/lib/python2.7/dist-packages
~/usr/local/lib/python2.7/dist-packages$ls -al
...
pydlt645-1.0.0-py2.7.egg
...

### Uninstall pydlt645 in Python-2.7 base on ubuntu16.04
Method1:
**~$sudo pip uninstall pydlt645**
(result1 and result2 above has been removed.)

Method2:
**~$sudo easy_install -m pydlt645**
(result1 has been removed and result2 has not been removed)

### Test pydlt645 
~$chmod 777 test.py
~$./test.py

## FAQ
Q1:SyntaxError: Non-ASCII character '\xe5' in file xxxx ?
A:In python, default encoding is ASCII.If the file has chinese, we should add **#-\*- coding: UTF-8 -\*-** in the header of it.

Q2:TypeError: unbound method getvalue() must be called with ReadDLT645RegistersRequest instance as first argument (got nothing instead) ?
A:Function getvalue() is not static method and must be called before instantiating, as following.
```ruby
obj = ReadDLT645RegistersRequest()
obj.getvalue()
```

or change getvalue() to static method, as following:

```ruby
class ReadDLT645RegistersRequest:
    @staticmethod
    def getvalue():
        ......

ReadDLT645RegistersRequest.getvalue()
```

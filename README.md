# PowerShell NamedPipeConnection module

This module implements a PowerShell custom remote connection based on named pipes.
It will connect to any local process that is running PowerShell which has a named pipe listener.
Currently, all versions of PowerShell 5.1 - 7.x run with a named pipe listener running by default.  

This module is intended for use in PowerShell testing, and as an example of how to use the new publicly available PowerShell APIs for creating custom connection and transport objects as outlined in this [RFC document](https://github.com/PowerShell/PowerShell-RFC/blob/master/Archive/Draft/RFC0063-Custom-Remote-Connections.md).

## Supported PowerShell versions

This module only works with PowerShell version 7.3 and greater.  

## New-NamedPipeSession

This command attempts to connect to a local process running PowerShell via a provided process Id, and return a `PSSession` object.
The `PSSession` object can be used with PowerShell core cmdlets to enter into an interactive session or invoke commands and script on the remote session.

```powershell
Import-Module -Name Microsoft.PowerShell.NamedPipeConnection
$session = New-NamedPipeSession -ProcessId 27536 -ConnectingTimeout 10 -Name MyConnect
$session

 Id Name            Transport ComputerName    ComputerType    State         ConfigurationName     Availability
 -- ----            --------- ------------    ------------    -----         -----------------     ------------
  1 MyConnect       PSNPTest  LocalMachine:2… RemoteMachine   Opened                                 Available

Enter-PSSession -Session $session
[LocalMachine:27536]: PS C:\> $PID
27536
[LocalMachine:27536]: PS C:\>
```

## Implementation

The NamedPipe custom connection is implemented in three main steps:

### One: Create a connection implementation

PowerShell remoting connections are defined by a connection type which is derived from the `System.Management.Automation.Runspaces.RunspaceConnectionInfo` class.
This new connection type is `NamedPipeInfo` and contains all properties needed to define a connection.
`NamedPipeInfo` class adds new properties specific to connecting to a process, but also inherits properties from its base class, some of which do not apply to process connections.  

The `NamedPipeInfo` class wraps PowerShell's `RemoteSessionNamedPipeClient` class that implements creating a named pipe client object which can connect to local process running PowerShell.
So the endpoint for this custom connection is just another instance of a local process running PowerShell.
This connection object can connect to such a local process by the process Id.  

A connection type derived from `RunspaceConnectingInfo` must override two base class methods in order to operate within the PowerShell remoting layer.

#### Clone

The `Clone` method override is used internally by PowerShell remoting to make a copy of the connection object.

#### CreateClientSessionTransportManager

The `CreateClientSessionTransportManager` method override is how PowerShell remoting creates the custom connection transport manager, in this case a `NamedPipeClientSessionTransportMgr` object implemented in this module.

### Two: Create a transport implementation

The transport manager object handles remoting protocol (PSRP) message routing over the connection.
It holds a reference to the connection object and creates a listener thread to listen for protocol messages from the target through the named pipe TextReader data stream.
The transport manager also writes protocol messages to the target via the named pipe TextWriter data stream.  

A custom transport implementation is created by deriving from the `System.Management.Automation.Remoting.Client.ClientSessionTransportManagerBase` abstract class.
There are three primary method overrides that must be implemented for a custom connection.

#### CreateAsync

The `CreateAsync` method override is called by PowerShell remoting internals to initiate a connection and then establish a remoting session through protocol messages.
For this named pipe transport, `CreateAsync` calls `RemoteSessionNamedPipeClient.Connect` to start a connection to the provided local process Id, and then starts a listening thread for the named pipe read data stream.
The named pipe reader data is directed to an internal handler (`HandleDataReceived`) that reads and responds to protocol messages from the target session.
Any connection errors are considered terminating errors and are passed to an internal PowerShell protocol error handler via the internal `RaiseErrorHandler` method.  
The named pipe writer data stream is set as the transport message writer through the `SetMessageWriter` base transport method, which allows PowerShell remoting internals to write protocol messages to the target session.  

Note that the `NamedPipeInfo` class includes a connecting timeout (`ConnectingTimeout`) property that will abort a connection attempt after a provided period of time, and which the wrapped `RemoteSessionNamedPipeClient` uses when its `Connect` method is called.
This is needed because PowerShell remoting protocol will wait indefinitely for a connection response after `CreateAsync` is called, and it is useful to have a time out abort for non-interactive use.

#### Dispose

The `Dispose` method is called by PowerShell remoting internals to ensure all resources are deallocated.
This named pipe implementation calls `CloseConnection`, which ensures connection objects are closed and disposed.

#### CleanupConnection

The `CleanupConnection` method is called by PowerShell remoting internals after a session close complete event.
It is needed because session termination may not be initiated by the client and may be terminated because of errors on the target or because of named pipe connection errors.
This method ensures connection resources are cleaned up by calling the `CloseConnection` method, regardless of how the connection is terminated.  

#### CloseAsync

The `CloseAsync` method override is optional but is not needed and not used by the named pipe transport manager.
It can be used to perform any needed work before a connection is closed from the client.

### Three: Create a cmdlet that starts and returns an open session

This custom named pipe remoting connection module exposes a single `New-NamedPipeSession` cmdlet.
This cmdlet creates a PowerShell remoting session (`PSSession`) based on a provided local host process Id.
The cmdlet has the following parameters:

#### ProcessId

The Id of a local process running PowerShell where a remoting connection will be made.

#### Name

Optional name for the created remote session (`PSSession`).

#### ConnectingTimeout

Optional time in seconds to wait for a connection to be established, after which the connection attempt is abandoned.

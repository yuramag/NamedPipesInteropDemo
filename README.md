<h1>.NET and Delphi Inter-Process Communication via Named Pipes</h1>

<h2>Introduction</h2>

<p>Let&#39;s imagine that we have legacy code written in Delphi that does some sort of sophisticated data processing. The code is 32-bit and not thread safe, meaning it can serve only one request at a time. Our task would be to extend functionality of the existing code by introducing a .NET wrapper around it, possibly exposing functionality via WCF, or some other means usable in the .NET world, and implement efficient communication mechanism between the two. We would also need to have our .NET wrapper to be 64-bit and potentially support parallel processing.</p>

<p>The latter two requirements dictate that both the wrapper and Delphi part must be compiled into separate executables, because the alternative of sharing single process boundaries (communicating via P/Invoke, for example) is not an option in this case: 32-bit DLL cannot live inside 64-bit process, and the parallelism cannot be achieved within single process because our Delphi code is not thread-safe.</p>

<p>There are many ways to organize inter-process communication (IPC). However, in this article, we will focus on standard Windows Named Pipes. The attached solution consists of two projects: .NET and Delphi console applications. In order to simulate data processing, we will implement 4 arithmetic operations in the Delphi project: addition, subtraction, multiplication, and division. The .NET application will serve the purpose of calling one of those 4 operations. Since we are focusing on Named Pipes inter-process communication aspect, we will not cover other issues like security, automatic message serialization, automatic call dispatch, or supporting parallelism.</p>

<h2>Message Format</h2>

<p>In the current solution, two processes communicate by sending string messages in the form of XML. Here is an example of message XML describing request of multiplying <code>X</code> and <code>Y</code> variables:</p>

<pre lang="xml">
&lt;Message Name=&quot;Mult&quot;&gt;
  &lt;Data Key=&quot;X&quot;&gt;<strong>3</strong>&lt;/Data&gt;
  &lt;Data Key=&quot;Y&quot;&gt;<strong>5</strong>&lt;/Data&gt;
&lt;/Message&gt;</pre>

<p>The resulting message would be:</p>

<pre lang="xml">
&lt;Message Name=&quot;Result&quot;&gt;
  &lt;Data Key=&quot;Result&quot;&gt;15&lt;/Data&gt;
&lt;/Message&gt;</pre>

<p>If calculation, for some reason, produces exception, the message would look as follows:</p>

<pre lang="xml">
&lt;Message IsError=&quot;1&quot; ErrorCode=&quot;EZeroDivide&quot; ErrorText=&quot;Floating point division by zero&quot;/&gt;</pre>

<p>Both the caller and callee must be aware of the format of messages and agree to follow the protocol. That said, we can describe any method with any number of arguments, send it through a pipe, and expect results back from the channel including any exceptions that might occur down the road. Although the format of messages is more verbose than it could be, it is acceptable for our demo purposes.</p>

<p>The message structure is encapsulated in both .NET and Delphi projects in <code>PipeMessage</code> and <code>TPipeMessage</code> classes respectively.</p>

<h2>High Level Protocol</h2>

<p>As mentioned above, there are two console applications in the solution. If we execute the Delphi application, it will not do anything unless we supply a command-line argument with the name of Pipe Channel as follows: <code>pipe=&lt;name_of_pipe_channel&gt;</code>. This name is an arbitrary string representing the name of the channel we expect to communicate through. If the channel name is specified, Delphi application will instantiate a Pipe Server and start listening for commands.</p>

<p>The calling .NET application, on the other hand, defines a channel named &quot;<code>interop.demo</code>&quot; and attempts to send all requests through that channel. After sending a request, it might either get a result, if Delphi application is already instantiated and listening to that channel, or an exception named <code>PipeServerNotFoundException</code>, indicating that no such channel exists yet. In the latter case while handling this exception, it will attempt to instantiate the Delphi executable passing the channel name as command-line argument, and reissue the same request one more time.</p>

<pre lang="cs">
public sealed class PipeInteropDispatcher
{
  private const string c_pipeName = @&quot;interop.demo&quot;;
  private const string c_pipeServerName = &quot;NamedPipesInteropDelphi.exe&quot;;

  private static async Task CreatePipeServer(string pipeName)
  {
    try
    {
      Process.Start(c_pipeServerName, &quot;pipe=&quot; + pipeName);
      await new PipeClient(pipeName).WaitForPipe(10000);
    }
    catch (Exception ex)
    {
      throw new InvalidOperationException(string.Format(
        &quot;Unable to instantiate Server: {0}&quot;, ex.Message), ex);
    }
  }

  public static async Task<pipemessage> ProcessRequestAsync(PipeMessage request)
  {
    var needInstance = false;
    var result = default (PipeMessage);
    var pipe = new PipeClient(c_pipeName);

    try
    {
      result = await pipe.ProcessRequest(request);
    }
    catch (PipeServerBrokenPipeException)
    {
      needInstance = true;
    }
    catch (PipeServerNotFoundException)
    {
      needInstance = true;
    }

    if (needInstance)
    {
      await CreatePipeServer(c_pipeName);
      result = await pipe.ProcessRequest(request);
    }

    return result;
  }
}</pipemessage></pre>

<p>With these rules in place, we gain the following benefits:</p>

<ol>
	<li>Ease of debugging - we can run Delphi application from Delphi Debugger specifying proper pipe channel name as command line argument.</li>
	<li>Delphi process can terminate itself (e.g. on idling timeout) at any time without breaking the system because any subsequent request coming through the pipe would fail with appropriate exception and the calling process would re-instantiate the app. In fact, we can forcibly terminate Delhi process from Task Manager at any time with the same effect.</li>
	<li>We can implement parallelism functionality on top of existing architecture (not presented in current solution) by maintaining a pool of pipe channel names that we can use to send messages trough in parallel fashion.</li>
</ol>

<h2>Client Side (.NET App)</h2>

<p>Depending on the user&#39;s input, we send an appropriate method name as well as <code>X</code> and <code>Y</code> variables to the following method:</p>

<pre lang="cs">
private static async Task&lt;decimal&gt; CalcAsync(string method, decimal x, decimal y)
{
  var args = new Dictionary&lt;string, string&gt;
  {
    {&quot;X&quot;, x.ToString(CultureInfo.InvariantCulture)},
    {&quot;Y&quot;, y.ToString(CultureInfo.InvariantCulture)}
  };
  var result = await PipeInteropDispatcher.ProcessRequestAsync(new PipeMessage(method, args));
  return decimal.Parse(result[&quot;Result&quot;]);
}</pre>

<p>We simply create <code>Dictionary&lt;string, string&gt;</code> object, populate it with input arguments, and hand it over to the <code>PipeInteropDispatcher</code> for further processing.</p>

<p><code>PipeClient</code> class plays central role in sending messages through the pipe channel. Internally, it is using standard .NET <code>NamedPipeClientStream</code> class to communicate with native Windows Named Pipes API. In order to efficiently transfer large messages over the channel, we are sending them in chunks of 4KB maximum. So every request larger than 4KB gets broken down into series of smaller messages that are sequentially being streamed out through the channel.</p>

<p>In order to coordinate original message size with counterpart process, the first 4 bytes of streaming data will contain an<code> Int32</code> number representing the size of that original message. This way, the process receiving the message will read the first 4 bytes from the stream and use that value to continue reading data until the entire X bytes of the message are read.</p>

<pre lang="cs">
public sealed class PipeClient
{
  private readonly string m_serverName;

  private readonly string m_pipeName;

  public PipeClient(string pipeName)
    : this(&quot;.&quot;, pipeName) { }

  public PipeClient(string serverName, string pipeName)
  {
    m_serverName = serverName;
    m_pipeName = pipeName;
  }

  private static async Task&lt;bool&gt; CopyStream(Stream src, Stream dst, int bytes, int bufferSize)
  {
    var buffer = new byte[bufferSize];
    int read;
    while (bytes &gt; 0 &amp;&amp; 
    (read = await src.ReadAsync(buffer, 0, Math.Min(buffer.Length, bytes))) &gt; 0)
    {
      await dst.WriteAsync(buffer, 0, read);
      bytes -= read;
    }
    return bytes == 0;
  }

  private async Task&lt;string&gt; ProcessRequest(string message)
  {
    var dataBuffer = Encoding.ASCII.GetBytes(message);
    var dataSize = dataBuffer.Length;

    if (dataSize == 0)
      return null;

    using (var pipe = new NamedPipeClientStream(m_serverName, m_pipeName, 
      PipeDirection.InOut, PipeOptions.Asynchronous | PipeOptions.WriteThrough))
    {
      try
      {
        await Task.Run(() =&gt; pipe.Connect(500));
      }
      catch (TimeoutException e)
      {
        throw new PipeServerNotFoundException(e.Message, e);
      }

      pipe.ReadMode = PipeTransmissionMode.Message;

      const int cBufferSize = 4096;

      try
      {
        var dataSizeBuffer = BitConverter.GetBytes(dataSize);
        await pipe.WriteAsync(dataSizeBuffer, 0, dataSizeBuffer.Length);
        using (var stream = new MemoryStream(dataBuffer))
          await CopyStream(stream, pipe, dataSize, cBufferSize);
        await pipe.FlushAsync();

        dataSizeBuffer = new byte[sizeof(Int32)];
        var bytesRead = await pipe.ReadAsync(dataSizeBuffer, 0, sizeof(Int32));
        if (bytesRead &lt;= 0)
          throw new PipeServerBrokenPipeException();

        dataSize = BitConverter.ToInt32(dataSizeBuffer, 0);
        if (dataSize &lt;= 0)
          return null;

        using (var stream = new MemoryStream(dataSize))
        {
          if (!await CopyStream(pipe, stream, dataSize, cBufferSize))
            throw new PipeServerBrokenPipeException();
          var resultBuffer = stream.GetBuffer();
          var decoder = Encoding.ASCII.GetDecoder();
          var charCount = decoder.GetCharCount(resultBuffer, 0, dataSize, false);
          var charResultBuffer = new char[charCount];
          decoder.GetChars(resultBuffer, 0, dataSize, charResultBuffer, 0, false);
          decoder.Reset();
          return new string(charResultBuffer);
        }
      }
      catch (IOException ex)
      {
        // Console.WriteLine(ex.Message);
        // NOTE: This is not reliable, but will do for now
        if (ex.Message.Contains(&quot;Pipe is broken&quot;))
          throw new PipeServerBrokenPipeException();
        throw;
      }
    }
  }

  public async Task&lt;PipeMessage&gt; ProcessRequest(PipeMessage message)
  {
    var resultMessage = await ProcessRequest(message.ToString());
    var result = new PipeMessage(resultMessage);
    if (result.IsError)
      throw new PipeServerException(result.ErrorCode, result.ErrorText);
    return result;
  }

  public async Task WaitForPipe(int timeout)
  {
    await Task.Run(() =&gt;
    {
      using (var pipe = new NamedPipeClientStream(m_serverName, m_pipeName, 
        PipeDirection.InOut, PipeOptions.Asynchronous | PipeOptions.WriteThrough))
        pipe.Connect(timeout);
    });
  }
}</pre>

<h2>Server Side (Delphi App)</h2>

<p>Here is the list of classes used on the server side to support Named Pipes communication:</p>

<ol>
	<li><code>TNamedPipeServer</code>. This class encapsulates low level interaction with native Windows Named Pipes API.</li>
	<li><code>TPipeMessage</code>. Represents an object wrapper around XML message.</li>
	<li><code>TCalcPipeServer</code>. Inherits from <code>TNamedPipeServer</code> and overrides its <code>DispatchMessage</code> method passing request to a Dispatcher.</li>
	<li><code>TCalcRequestDispatcher</code>. Encapsulates method dispatching functionality converting incoming XML messages into actual method calls, and packing results back to resulting XML messages.</li>
	<li><code>TCalcProcessor</code>. A helper class containing actual implementation of business logic (4 arithmetic operations in our case).</li>
</ol>

<p>When Delphi App starts, it will read the pipe channel name from command line and instantiate a Pipe Server passing that channel name to it. The Pipe Server instance will create the channel and start listening to incoming requests until user presses <code>ENTER</code> or application gets shut down some other way.</p>

<p>Upon receiving a message from channel, the Pipe Server will hand it over to a <code>DispatchMessage</code> virtual method, which is overridden by <code>TCalcPipeServer</code>:</p>

<pre lang="text">
procedure TCalcPipeServer.DispatchMessage(ARequestStream, AResponseStream: TStream);
var
  lData: string;
begin
  SetLength(lData, ARequestStream.Size);
  ARequestStream.Read(lData[1], Length(lData));
  lData := RequestDispatcher.ProcessRequest(lData);
  AResponseStream.Write(lData[1], Length(lData));
end;</pre>

<p><code>RequestDispatcher</code> in this case is represented by <code>TCalcRequestDispatcher</code> class that transforms XML requests into actual method calls and returns result back as XML string:</p>

<pre lang="text">
function TCalcRequestDispatcher.InternalProcessRequest(const ARequest: string): string;
var
  lRequest: TPipeMessage;
  X, Y, lResult: Double;
begin
  Result := &#39;&#39;;
  lRequest := TPipeMessage.Create(ARequest);
  try
    if lRequest.Name = &#39;Add&#39; then
    begin
      X := StrToFloat(lRequest.Data[&#39;X&#39;]);
      Y := StrToFloat(lRequest.Data[&#39;Y&#39;]);
      lResult := TCalcProcessor.Add(X, Y);
      Result := TPipeMessage.MakeMessage(&#39;Result&#39;, [&#39;Result&#39;], [FloatToStr(lResult)]);
    end
    else if lRequest.Name = &#39;Subtract&#39; then
    begin
      X := StrToFloat(lRequest.Data[&#39;X&#39;]);
      Y := StrToFloat(lRequest.Data[&#39;Y&#39;]);
      lResult := TCalcProcessor.Subtract(X, Y);
      Result := TPipeMessage.MakeMessage(&#39;Result&#39;, [&#39;Result&#39;], [FloatToStr(lResult)]);
    end
    else if lRequest.Name = &#39;Mult&#39; then
    begin
      X := StrToFloat(lRequest.Data[&#39;X&#39;]);
      Y := StrToFloat(lRequest.Data[&#39;Y&#39;]);
      lResult := TCalcProcessor.Mult(X, Y);
      Result := TPipeMessage.MakeMessage(&#39;Result&#39;, [&#39;Result&#39;], [FloatToStr(lResult)]);
    end
    else if lRequest.Name = &#39;Div&#39; then
    begin
      X := StrToFloat(lRequest.Data[&#39;X&#39;]);
      Y := StrToFloat(lRequest.Data[&#39;Y&#39;]);
      lResult := TCalcProcessor.Divide(X, Y);
      Result := TPipeMessage.MakeMessage(&#39;Result&#39;, [&#39;Result&#39;], [FloatToStr(lResult)]);
    end
    else if lRequest.Name = &#39;ExitProcess&#39; then
    begin
      ExitProcess(0);
    end
    else if lRequest.Name = &#39;Ping&#39; then
    begin
      // do nothing;
    end
    else
      raise Exception.CreateFmt(&#39;Unknown request: %s&#39;, [lRequest.Name]);
  finally
    lRequest.Free;
  end;
end;</pre>

<h2>Pipe Server Implementation</h2>

<p><code>TNamedPipeServer</code> class accepts a string argument in its constructor representing pipe channel name argument and internally creates a <code>Listener Thread</code>, a <code>Message Queue</code>, and a <code>Worker Thread</code>.</p>

<p>The Listener is represented by <code>TPipeListenerThread</code> class. Its main job is to constantly monitor the Pipe Channel for incoming requests, and sending them over to the Message Queue. The latter is represented by <code>TThreadQueue</code> class, a thread-safe implementation of a Queue with blocking behavior, meaning that the pushing thread will get blocked if the queue size has reached maximum number of messages, and the popping thread will get blocked if there are no elements in the queue. And finally, the Working Thread is represented by <code>TPipeWorkerThread</code> class, which is continuously extracting messages from the Queue and handing them over to the Message Handler.</p>

<p>The Message Handler is represented by <code>TPipeMessageHandler</code> class and implements stream read/write logic. The Message Handler uses given handle to the Pipe Channel and starts reading from it. Once the message is read, it is being passed to the Dispatcher (that we already described), and the final result is being written back to the stream according to the protocol rules. The source code of message streaming is presented below:</p>

<pre lang="text">
procedure TPipeMessageHandler.Execute;
var
  lReadStream, lWriteStream: TMemoryStream;
  lBytesRead, lBytesWritten: DWORD;
  lSuccess: BOOL;
  lDataSize, lBufferSize: Integer;
  lBuffer: PChar;
begin
  lBufferSize := FServer.BufferSize;
  GetMem(lBuffer, lBufferSize);
  lReadStream := TMemoryStream.Create;
  try
    try
      lSuccess := ReadFile(fPipeHandle, lDataSize, SizeOf(lDataSize), lBytesRead, nil);

      if not lSuccess then
      begin
        if GetLastError = ERROR_BROKEN_PIPE then
          Exit
        else
          RaiseLastOSError;
      end;

      while lDataSize &gt; 0 do
      begin
        lSuccess := ReadFile(fPipeHandle, lBuffer^, lBufferSize, lBytesRead, nil);

        if not lSuccess and (GetLastError &lt;&gt; ERROR_MORE_DATA) then
          RaiseLastOSError;

        if lBytesRead &gt; 0 then
          lReadStream.Write(lBuffer^, lBytesRead);

        Dec(lDataSize, lBytesRead);
      end;

      if not lSuccess then
        RaiseLastOSError;

      if lReadStream.Size &gt; 0 then
      begin
        lReadStream.Position := 0;
        lWriteStream := TMemoryStream.Create;
        try
          FServer.DispatchMessage(lReadStream, lWriteStream);
          lReadStream.Position := 0;
          lReadStream.SetSize(0);
          lWriteStream.Position := 0;

          lDataSize := lWriteStream.Size;
          lSuccess := WriteFile(fPipeHandle, lDataSize, SizeOf(lDataSize), lBytesWritten, nil);

          if not lSuccess then
            RaiseLastOSError;

          while lDataSize &gt; 0 do
          begin
            lBytesRead := lWriteStream.Read(lBuffer^, lBufferSize);
            Dec(lDataSize, lBytesRead);
            lSuccess := WriteFile(fPipeHandle, lBuffer^, lBytesRead, lBytesWritten, nil);

            if not lSuccess then
              RaiseLastOSError;
          end;
        finally
          lWriteStream.Free;
        end;
      end;
    except
      on E: EOSError do
      begin
        if not (E.ErrorCode in [ERROR_NO_DATA, ERROR_BROKEN_PIPE]) then
          raise;
      end;
    end;
  finally
    FreeMem(lBuffer);
    lReadStream.Free;
    FlushFileBuffers(FPipeHandle);
    DisconnectNamedPipe(FPipeHandle);
    CloseHandle(FPipeHandle);
  end;
end;</pre>

<h2>Summary</h2>

<p>In this article, I tried to present one of the possible solutions of implementing inter-process communication between .NET Framework and legacy projects that might have certain restrictions, like thread-safety or not being able to run within 64-bit process. Sometimes it is very important to be able to initiate graceful migration from legacy code bases to .NET Framework without introducing major side effects and making transition as smooth as possible. Inter-process communication based on Named Pipes offers very efficient bi-directional protocol that might help in achieving those goals.</p>

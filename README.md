# BOA分析

## BOA流程图示意





* `write()`函数返回-1的几种判断情况:
    * EINTR：被信号中断
    * “Resource temporarily unavailable.” The call might work if you try again later. The macro `EWOULDBLOCK` is another name for `EAGAIN`; they are always the same in the GNU C Library.

```C
bytes_read = read(req->data_fd, req->header_end, bytes_to_read);
if (bytes_read == -1)
{
    if (errno == EINTR)	//被信号中断
        return 1;
    else if (errno == EWOULDBLOCK || errno == EAGAIN)	// 管道阻塞
        return -1;          /* request blocked at the pipe level, but keep going */
    else
    {
        req->status = DEAD;
        log_error_doc(req);
        perror("pipe read");
        return 0;
    }
}
```





## process_request 对返回值的处理



``` c
switch (retval)
		{
		case -1: 	// 请求放入阻塞队列中
		case 0:		// 处理完成，删除资源
		case 1:		// 下次再进行处理
		default:	// 状态更改为DEAD，
		}
```















## PIPE.C



* function：`int read_from_pipe(request* req)`
* 该函数用来读取子进程传入PIPE中的数据（`HTTP消息体的数据`），一共有如下几种情况：
    * 读取阻塞，
    * 读取被信号打断
    *

``` C
int read_from_pipe(request* req)
{
	int bytes_read; /* signed */
	unsigned int bytes_to_read = BUFFER_SIZE - (req->header_end - req->buffer - 1);

	// request中装载数据的buf满了
	if (bytes_to_read == 0)
	{
		if (req->cgi_status == CGI_PARSE)
		{
			req->cgi_status = CGI_BUFFER;
			*req->header_end = '\0';
			return process_cgi_header(req); /* cgi_status will change */
		}
		req->status = PIPE_WRITE;
		return 1;
	}
	// request中装载数据的仍然有空余,读取管道中的数据
	bytes_read = read(req->data_fd, req->header_end, bytes_to_read);
	if (bytes_read == -1)	// 读取失败
	{
		if (errno == EINTR)	//被信号中断，下次再进行处理
			return 1;
		else if (errno == EWOULDBLOCK || errno == EAGAIN)// 读取管道被堵塞，放入堵塞队列中
			return -1;    
		else
		{
			req->status = DEAD;
			log_error_doc(req);
			perror("pipe read");
			return 0;			// 读取失败，返回值0，外面会删除这个请求的信息
		}
	}
	// 因为每次读取出来的数据都加了'\0'，所以下面调用printf输出的时候，数据会输出不完整。
	*(req->header_end + bytes_read) = '\0';

	if (bytes_read == 0)	// 读取PIPE到文件的结束位置
	{
		req->status = PIPE_WRITE;
		if (req->cgi_status == CGI_PARSE)// 如果还没有创建HTTP头，则进行创建
		{
			req->cgi_status = CGI_DONE;	// 状态修改为HTTP头创建完成。
			*req->header_end = '\0'; /* points to end of read data */
			return process_cgi_header(req); /* cgi_status will change */
		}
		req->cgi_status = CGI_DONE;
		return 1;
	}

	req->header_end += bytes_read;		// 偏移结束位置到实际的结束位置

	if (req->cgi_status != CGI_PARSE)
		return write_from_pipe(req); /* why not try and flush the buffer now? */
	else
	{
		char* c, * buf;
		buf = req->header_line;
        // 如果没有读取到换行，则放到下一次进行读取
		c = strstr(buf, "\n\r\n");		
		if (c == NULL)
		{
			c = strstr(buf, "\n\n");
			if (c == NULL)
			{
				return 1;
			}
		}
        // 如果已经读取到换行，则进行添加HTTP HEADER
		req->cgi_status = CGI_DONE;
		*req->header_end = '\0'; /* points to end of read data */
		return process_cgi_header(req); /* cgi_status will change */
	}
	return 1;
}
```




.. _osservices_formated_output_userapi_backend:

Zephyr格式化输出-用户API和后端
###############################


在Zephyr中提供以下API提供格式化输出

* printk: 内核调试信息打印
* \*printf：printf/fprintf/vfprintf/vprintf等标准C输出
* shell_print: Shell系统打印
* LOG\_\*: LOG_INF/LOG_DBG/LOG_ERR/LOG_WAR,LOG系统打印


以下分别说明其格式化处理函数及用什么后端输出，也可以直接跳到最后看总结。

prink
======

函数 ``printk`` 通常用于格式化输出内核调试信息的打印，为了方便我们通常也会在自己写的应用中调用该API进行打印输出。


格式化处理函数
~~~~~~~~~~~~~~~

函数 ``printk`` 源代码在 ``lib/os/printk.c`` 中通过调用 ``vprintk`` 完成

.. code:: c

	void vprintk(const char *fmt, va_list ap)
	{
		struct out_context ctx = { 0 };

		cbvprintf(char_out, &ctx, fmt, ap);
	}


\ ``vprintk`` 使用 ``cbvprintf`` 完成字符串格式化，并同时使用 ``char_out`` 一个一个字符输出

.. code:: c

	static int char_out(int c, void *ctx_p)
	{
		struct out_context *ctx = ctx_p;

		ctx->count++;
		return _char_out(c);
	}


后端
~~~~~

\ ``char_out`` 由 ``__printk_hook_install`` 注册的 ``_char_out`` 进行字符输出

.. code:: c

	void __printk_hook_install(int (*fn)(int))
	{
		_char_out = fn;
	}


\ ``__printk_hook_install`` 会在zephyr console初始化的时候调用，设置入console的输出，根据平台硬件的不同zephyr的console支持UART，RAM, RTT, IPM等等。一般情况下我们默认会使用UART，也就是printk输出到串口。
console的代码在 ``driver/console/`` 下，串口的文件是 ``uart_console.c`` 在其初始化函数 ``uart_console_init`` 中调用 ``uart_console_hook_install`` 进行printk hook注册

.. code:: c

	static void uart_console_hook_install(void)
	{
	#if defined(CONFIG_STDOUT_CONSOLE)
		__stdout_hook_install(console_out);
	#endif
	#if defined(CONFIG_PRINTK)
		__printk_hook_install(console_out);
	#endif
	}


\*printf
=========

Zephyr在 ``lib/libc/minimal/source/stdout`` 下实现了 ``printf/fprintf/vfprintf/vprintf`` 等标准C输出, 以在 ``fprint.c`` 中的 ``printf`` 进行说明

格式化处理函数
~~~~~~~~~~~~~~~

.. code:: c

	int printf(const char *ZRESTRICT format, ...)
	{
		va_list vargs;
		int     r;

		va_start(vargs, format);
		r = cbvprintf(fputc, DESC(stdout), format, vargs);
		va_end(vargs);

		return r;
	}


\ ``printf`` 使用 ``cbvprintf`` 完成字符串格式化，并同时使用 ``fputc`` 一个一个字符输出, 在 ``stdout_console.c`` 中

.. code:: c

	int fputc(int c, FILE *stream)
	{
		return zephyr_fputc(c, stream);
	}


**注意，当Zephyr使用第三方libc时，例如newlib，库中的标准格式化输出函数将使用第三方库中格式化处理而不是使用 cbvprintf。**

后端
~~~~~

\ ``zephyr_fputc`` 实现为在 ``z_impl_zephyr_fputc``

.. code:: c

	int z_impl_zephyr_fputc(int c, FILE *stream)
	{
		return (stream == stdout || stream == stderr) ? _stdout_hook(c) : EOF;
	}

其hook函数由``__stdout_hook_install注册

.. code:: c

	void __stdout_hook_install(int (*hook)(int))
	{
		_stdout_hook = hook;
	}


和 ``printk`` 一样其hook函数会在console driver中调用 ``__stdout_hook_install`` 进行注册(前面代码 ``uart_console_hook_install`` 可以看到)

shell_print
==============

shell系统要控制自己的输入输出，当同为一个后端时，例如都在串口上，为了不被 ``printk`` 和 ``printf`` 干扰，建议使用 ``shell_print`` 进行打印。
\ ``shell_print`` 是一个 ``include\shell\shell.h`` 中的宏，实际使用的是 ``subsys\shell\shell.c`` 中的 ``shell_fprintf`` 为简化说明，列出调用关系:
\ ``shell_print->shell_fprintf->shell_vfprintf->z_shell_vfprintf->z_shell_fprintf_fmt`` 。

格式化处理函数
~~~~~~~~~~~~~~

在 ``shell_fprintf.c`` 中实现的 ``z_shell_fprintf_fmt``

.. code:: c

	void z_shell_fprintf_fmt(const struct shell_fprintf *sh_fprintf,
			 const char *fmt, va_list args)
	{
		(void)cbvprintf(out_func, (void *)sh_fprintf, fmt, args);

		if (sh_fprintf->ctrl_blk->autoflush) {
			z_shell_fprintf_buffer_flush(sh_fprintf);
		}
	}


\ ``z_shell_fprintf_fmt`` 使用 ``cbvprintf`` 完成字符串格式化，并同时使用 ``out_func`` 一个一个字符输出。

.. code:: c

	static int out_func(int c, void *ctx)
	{
		const struct shell_fprintf *sh_fprintf;
		const struct shell *shell;

		sh_fprintf = (const struct shell_fprintf *)ctx;
		shell = (const struct shell *)sh_fprintf->user_ctx;

		if ((shell->shell_flag == SHELL_FLAG_OLF_CRLF) && (c == '\n')) {
			(void)out_func('\r', ctx);
		}
		//装入buffer
		sh_fprintf->buffer[sh_fprintf->ctrl_blk->buffer_cnt] = (uint8_t)c;
		sh_fprintf->ctrl_blk->buffer_cnt++;

		//装满后才进行真正的flush写到后端
		if (sh_fprintf->ctrl_blk->buffer_cnt == sh_fprintf->buffer_size) {
			z_shell_fprintf_buffer_flush(sh_fprintf);
		}

		return 0;
	}

	void z_shell_fprintf_buffer_flush(const struct shell_fprintf *sh_fprintf)
	{
		//写到后端
		sh_fprintf->fwrite(sh_fprintf->user_ctx, sh_fprintf->buffer,
				sh_fprintf->ctrl_blk->buffer_cnt);
		sh_fprintf->ctrl_blk->buffer_cnt = 0;
	}


\ ``sh_fprintf->fwrite`` 是 ``SHELL_DEFINE->Z_SHELL_FPRINTF_DEFINE`` 注册函数 ``z_shell_print_stream`` 最后会调用到 ``z_shell_write``

.. code:: c

	void z_shell_write(const struct shell *shell, const void *data,
			size_t length)
	{
		__ASSERT_NO_MSG(shell && data);

		size_t offset = 0;
		size_t tmp_cnt;

		while (length) {
			int err = shell->iface->api->write(shell->iface,
					&((const uint8_t *) data)[offset], length,
					&tmp_cnt);
			(void)err;
			__ASSERT_NO_MSG(err == 0);
			__ASSERT_NO_MSG(length >= tmp_cnt);
			offset += tmp_cnt;
			length -= tmp_cnt;
			if (tmp_cnt == 0 &&
				(shell->ctx->state != SHELL_STATE_PANIC_MODE_ACTIVE)) {
				shell_pend_on_txdone(shell);
			}
		}


这里的 ``shell->iface->api->write`` 就是shell的后端write

后端
~~~~~~

shell的后端的所有实现都放在 ``subsys/shell/backends`` 下，支持uart, rtt, telnet, dummy，当选择串口作为后端时 ``shell_print`` 将输出到串口，串口后端实现的代码是shell_uart.c

.. code:: c

	const struct shell_transport_api shell_uart_transport_api = {
		.init = init,
		.uninit = uninit,
		.enable = enable,
		.write = write,
		.read = read,
	#ifdef CONFIG_MCUMGR_SMP_SHELL
		.update = update,
	#endif /* CONFIG_MCUMGR_SMP_SHELL */
	};

	static int write(const struct shell_transport *transport,
			const void *data, size_t length, size_t *cnt)
	{
		const struct shell_uart *sh_uart = (struct shell_uart *)transport->ctx;
		const uint8_t *data8 = (const uint8_t *)data;

		//使用串口直接输出
			for (size_t i = 0; i < length; i++) {
				uart_poll_out(sh_uart->ctrl_blk->dev, data8[i]);
			}

			*cnt = length;

			sh_uart->ctrl_blk->handler(SHELL_TRANSPORT_EVT_TX_RDY,
						sh_uart->ctrl_blk->context);


		return 0;
	}


LOG\_\*
=========

\ ``LOG_INF/LOG_DBG/LOG_ERR/LOG_WAR`` 是LOG系统打印，Zephyr提供这些格式化打印接口方便过滤和控制打印。
其调用关系可简化为：
\ ``LOG_\*->Z_LOG->Z_LOG2-Z_LOG_MSG2_CREATE->Z_LOG_MSG2_CREATE2``
\ ``Z_LOG_MSG2_CREATE2`终于会根据配置的不同调用`z_log_msg2_runtime_create`` 或 ``Z_LOG_MSG2_SIMPLE_CREATE`` 或 ``Z_LOG_MSG2_STACK_CREATE``

格式化处理函数
~~~~~~~~~~~~~~

*动态生成*

\ ``z_log_msg2_runtime_create->z_log_msg2_runtime_vcreate->z_impl_z_log_msg2_runtime_vcreate``

.. code:: c

	void z_impl_z_log_msg2_runtime_vcreate(uint8_t domain_id, const void *source,
					uint8_t level, const void *data, size_t dlen,
					const char *fmt, va_list ap)
	{
		int plen;

		if (fmt) {
			va_list ap2;

			va_copy(ap2, ap);
			plen = cbvprintf_package(NULL, Z_LOG_MSG2_ALIGN_OFFSET, 0,
						fmt, ap2);
			__ASSERT_NO_MSG(plen >= 0);
			va_end(ap2);
		} else {
			plen = 0;
		}

		size_t msg_wlen = Z_LOG_MSG2_ALIGNED_WLEN(plen, dlen);
		struct log_msg2 *msg;
		struct log_msg2_desc desc =
			Z_LOG_MSG_DESC_INITIALIZER(domain_id, level, plen, dlen);

		if (IS_ENABLED(CONFIG_LOG2_MODE_IMMEDIATE)) {
			msg = alloca(msg_wlen * sizeof(int));
		} else {
			msg = z_log_msg2_alloc(msg_wlen);
		}

		if (msg && fmt) {
			plen = cbvprintf_package(msg->data, (size_t)plen, 0, fmt, ap);
			__ASSERT_NO_MSG(plen >= 0);
		}

		z_log_msg2_finalize(msg, source, desc, data);
	}


使用 ``cbvprintf_package`` 打包格式化，使用 ``z_log_msg2_finalize`` 对打包后的数据进行输出

*静态生成*

\ ``Z_LOG_MSG2_SIMPLE_CREATE`` 先使用 ``CBPRINTF_STATIC_PACKAGE`` 打包格式化，再使用 ``z_log_msg2_finalize`` 对打包后的数据进行输出
\ ``Z_LOG_MSG2_STACK_CREATE``先使用 ``CBPRINTF_STATIC_PACKAGE`` 打包格式化，再通过 ``z_log_msg2_static_create->z_impl_z_log_msg2_static_create->z_log_msg2_finalize`` 对打包后的数据进行输出

后端
~~~~~

\ ``z_log_msg2_finalize`` 只是将 ``cbvprintf_package`` 或 ``CBPRINTF_STATIC_PACKAGE`` 打包后的数据送到log core， log core会将包送给后端进行显示。
log的backend实现文件放到 ``subsys\logging\`` 中以名字为 ``log_backend_\*.c`` , log系统的backend可以根据硬件平台的不同选择uart, rtt, swo, fs, net等等。其中uart实现在 ``log_backend_uart.c``
显示的执行流程是 ``process->log_output_msg2_process`` 简化如下

.. code:: c

	void log_output_msg2_process(const struct log_output *output,
					struct log_msg2 *msg, uint32_t flags)
	{
		//读取包数据
		uint8_t *data = log_msg2_get_package(msg, &len);

		if (len) {
			int err = cbpprintf(raw_string ? cr_out_func :  out_func,
						(void *)output, data);

			(void)err;
			__ASSERT_NO_MSG(err >= 0);
		}

		//使用cbpprintf解析包数据，并使用out_func输出
		if (len) {
			int err = cbpprintf(raw_string ? cr_out_func :  out_func,
						(void *)output, data);

			(void)err;
			__ASSERT_NO_MSG(err >= 0);
		}
	}


\ ``out_func`` 实现在log_output.c中，收到字符会先放到buffer，达到一定量后调用 ``log_output_flush->buffer_write->(output->func)`` 进行输出
output使用的是 ``LOG_OUTPUT_DEFINE(log_output_uart, char_out, uart_output_buf, sizeof(uart_output_buf))`` uart的 ``char_out`` 实现如下

.. code:: c

	static int char_out(uint8_t *data, size_t length, void *ctx)
	{
		ARG_UNUSED(ctx);
		int err;

		if (IS_ENABLED(CONFIG_LOG_BACKEND_UART_OUTPUT_DICTIONARY_HEX)) {
			dict_char_out_hex(data, length);
			return length;
		}

		if (!IS_ENABLED(CONFIG_LOG_BACKEND_UART_ASYNC) || in_panic || !use_async) {
			for (size_t i = 0; i < length; i++) {
				uart_poll_out(uart_dev, data[i]);
			}

			return length;
		}

		err = uart_tx(uart_dev, data, length, SYS_FOREVER_US);
		__ASSERT_NO_MSG(err == 0);

		err = k_sem_take(&sem, K_FOREVER);
		__ASSERT_NO_MSG(err == 0);

		(void)err;

		return length;
	}

可以看到是使用的串口驱动直接输出。

总结
======

printk: 使用cbvprintf完成字符串格式化，输出由console决定
\*printf: 当使用zephyr自己的minilibc时，使用cbvprintf完成字符串格式化，输出由console决定
shell_print: 使用 ``cbvprintf`` 完成字符串格式化，输出由shell自己配置的后端决定
LOG\_\*：使用 ``cbvprintf_package`` 或 ``CBPRINTF_STATIC_PACKAGE`` 打包格式化字符串，由 ``cbpprintf`` 根据包数据完成字符串格化，输出由log自己配置的后端决定

当console和shell后端还有log后端都选择为串口时，由于大家最后都是通过串口驱动输出，以上4类格式化API同时在多线程或中断中存在时会相互干扰，使用时需要留意。

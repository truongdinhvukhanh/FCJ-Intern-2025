> **📖 Bài viết gốc**: [How to automatically disable users in AWS Managed Microsoft AD based on GuardDuty findings](https://aws.amazon.com/blogs/security/how-to-automatically-disable-users-in-aws-managed-microsoft-ad-based-on-guardduty-findings/)
> 
> **👤 Tác giả**: Tim Kingdon
> 
> **📅 Ngày xuất bản**: 28-07-2025
> 
> **🌐 Nguồn**: AWS Security Blog
> 
> **👨‍💻 Người dịch**: Trương Đình Vũ Khanh - FCJ Intern
> 
> **📅 Ngày dịch**: 30-07-2025
> 
> **⏱️ Thời gian đọc**: 30 phút

---

## 📋 Tóm tắt

Các tổ chức đang đối mặt với số lượng mối đe dọa bảo mật ngày càng tăng, đặc biệt là dưới dạng các tài khoản người dùng bị xâm nhập. Việc giám sát và xử lý thủ công các hoạt động đáng ngờ không chỉ tốn thời gian mà còn dễ xảy ra lỗi của con người. Thiếu các phản ứng tự động đối với các sự cố bảo mật có thể dẫn đến những hậu quả thảm khốc, chẳng hạn như vi phạm dữ liệu và mất mát tài chính.

Bài đăng này trình bày cách phát hiện các sự kiện đáng ngờ bằng Amazon GuardDuty và tạo quy trình tự động hóa từ các phát hiện đó để vô hiệu hóa tài khoản người dùng trong AWS Directory Service for Microsoft Active Directory. Bài viết giải quyết các tình huống như khi một máy chủ web sử dụng tài khoản người dùng Microsoft Active Directory (tài khoản dịch vụ) để truy cập tài nguyên ứng dụng hoặc cơ sở dữ liệu trên các máy chủ khác, và bạn muốn tự động vô hiệu hóa tài khoản người dùng đó nếu phát hiện hoạt động đáng ngờ.

Hướng dẫn này sẽ chỉ cho bạn cách triển khai Microsoft Active Directory trong AWS Directory Services, thiết lập GuardDuty để giám sát các phiên bản Amazon Elastic Compute Cloud (Amazon EC2), và cấu hình Amazon EventBridge với AWS Step Functions để kích hoạt AWS Systems Manager Run Command nhằm lấy tên người dùng và vô hiệu hóa người dùng trong Active Directory. Cuối cùng, bài viết cũng cung cấp hướng dẫn kiểm tra giải pháp bằng cách mô phỏng mối đe dọa để xem quy trình tự động hóa hoạt động như thế nào.

**🎯 Đối tượng đọc**: Chuyên gia bảo mật, quản trị viên CNTT, người dùng AWS quan tâm đến bảo mật và tự động hóa.

**📊 Độ khó**: Nâng cao

**🏷️ Tags**: Amazon GuardDuty, GuardDuty, Bảo mật đám mây, Tự động hóa bảo mật, AWS Managed Microsoft AD, Phản ứng sự cố

---

## 📚 Mục lục
- [Cách tự động vô hiệu hóa người dùng trong AWS Managed Microsoft AD dựa trên các phát hiện của GuardDuty](#cách-tự-động-vô-hiệu-hóa-người-dùng-trong-aws-managed-microsoft-ad-dựa-trên-các-phát-hiện-của-guardduty)
  - [Solution overview](#solution-overview)
  - [GuardDuty](#guardduty)
  - [AWS Managed Microsoft AD](#aws-managed-microsoft-ad)
  - [Directory administration EC2 instance](#directory-administration-ec2-instance)
    - [Create a test Active Directory user](#create-a-test-active-directory-user)
    - [Create a privileged domain service account](#create-a-privileged-domain-service-account)
    - [Cấu hình Systems Manager](#cấu-hình-systems-manager)
  - [Systems Manager Run Command](#systems-manager-run-command)
  - [Step Functions](#step-functions)
  - [EventBridge](#eventbridge)
  - [Tạo một test EC2 instance](#tạo-một-test-ec2-instance)
  - [Kiểm thử](#kiểm-thử)
  - [Kết luận](#kết-luận)
  - [📖 Glossary - Thuật ngữ](#-glossary---thuật-ngữ)
  - [🔗 Tài liệu tham khảo](#-tài-liệu-tham-khảo)
    - [Tài liệu gốc](#tài-liệu-gốc)
    - [Tài liệu tiếng Việt](#tài-liệu-tiếng-việt)
    - [Tools và Services](#tools-và-services)
  - [💬 Ghi chú của người dịch](#-ghi-chú-của-người-dịch)
    - [Challenges trong quá trình dịch](#challenges-trong-quá-trình-dịch)
    - [Insights gained](#insights-gained)
  - [🤝 Đóng góp và Feedback](#-đóng-góp-và-feedback)

---

# Cách tự động vô hiệu hóa người dùng trong AWS Managed Microsoft AD dựa trên các phát hiện của GuardDuty

Các tổ chức đang đối mặt với số lượng mối đe dọa bảo mật ngày càng tăng, đặc biệt là dưới dạng các tài khoản người dùng bị xâm nhập. Việc giám sát và xử lý thủ công các hoạt động đáng ngờ không chỉ tốn thời gian mà còn dễ xảy ra lỗi của con người. Thiếu các phản ứng tự động đối với các sự cố bảo mật có thể dẫn đến những hậu quả thảm khốc, chẳng hạn như vi phạm dữ liệu và mất mát tài chính.

Trong bài đăng blog này, tôi sẽ chỉ cho bạn cách phát hiện các sự kiện đáng ngờ bằng [Amazon GuardDuty](https://aws.amazon.com/guardduty) và tạo một quy trình tự động hóa từ các phát hiện đó để vô hiệu hóa tài khoản người dùng trong [AWS Directory Service for Microsoft Active Directory](https://aws.amazon.com/directoryservice/).

Bài đăng này giải quyết các tình huống mà ví dụ như bạn có một web server sử dụng tài khoản người dùng Microsoft Active Directory (service account) để truy cập application hoặc database resources trên các server khác, và bạn muốn tự động vô hiệu hóa user account đó nếu phát hiện hoạt động đáng ngờ.

Tôi sẽ hướng dẫn bạn cách triển khai Microsoft Active Directory trong AWS Directory Services, thiết lập GuardDuty để giám sát các [Amazon Elastic Compute Cloud (Amazon EC2)](https://aws.amazon.com/ec2) instances, và cấu hình [Amazon EventBridge](https://aws.amazon.com/eventbridge) với [AWS Step Functions](https://aws.amazon.com/step-functions) để kích hoạt [AWS Systems Manager](https://aws.amazon.com/systems-manager) Run Command nhằm lấy username và vô hiệu hóa user trong Active Directory.

## Solution overview

Trong ví dụ này, được minh họa trong Hình 1, bạn triển khai một test EC2 instance và bật tính năng giám sát Runtime Monitoring của GuardDuty. Findings sẽ kích hoạt một EventBridge rule thực thi một Step Functions state machine, quy tắc này chạy hai Systems Manager Run Command documents để phát hiện username và vô hiệu hóa user đó bằng cách sử dụng directory administration EC2 instance.

![Figure 1: Solution architecture](/images/Figure-1-1.png)

Hình 1: Kiến trúc giải pháp

## GuardDuty

GuardDuty là một threat detection service tự động liên tục giám sát các hoạt động đáng ngờ và hành vi trái phép để bảo vệ AWS accounts, workloads và data của bạn được lưu trữ trong [Amazon Simple Storage Service (Amazon S3)](https://aws.amazon.com/s3).

Để kích hoạt GuardDuty:

1.  Truy cập GuardDuty trên AWS Management Console.
    1.  Nếu bạn đang kích hoạt GuardDuty lần đầu tiên, trong phần **Try threat detection with GuardDuty**, chọn **All Features** và sau đó chọn **Get Started**.
    2.  Nếu bạn đã sử dụng GuardDuty trước đây, chọn **Runtime Monitoring** và sau đó chọn **Enable** trong phần **Runtime Monitoring**.

    ![Figure 2: GuardDuty Runtime Monitoring enabled with EC2 monitoring](/images/Figure-2-1.png)

    Hình 2: GuardDuty Runtime Monitoring được bật với EC2 monitoring

## AWS Managed Microsoft AD

AWS Managed Microsoft AD cung cấp một dịch vụ được quản lý hoàn toàn cho Microsoft Active Directory (AD) trong AWS Cloud. Khi bạn tạo directory của mình, AWS triển khai hai domain controllers ở các Availability Zones riêng biệt dành riêng cho bạn để đảm bảo high availability. Đối với các use cases yêu cầu khả năng resilience và performance cao hơn nữa trong một AWS Region cụ thể hoặc trong những giờ cụ thể, bạn có thể scale AWS Managed Microsoft AD bằng cách triển khai thêm các domain controllers để đáp ứng nhu cầu của mình. Các domain controllers này có thể giúp load balance, tăng overall performance hoặc cung cấp các nodes bổ sung để bảo vệ chống lại các temporary availability issues. Sử dụng AWS Managed Microsoft AD, bạn có thể define số lượng domain controllers chính xác cho directory của mình dựa trên use case.

**Để deploy một AWS Managed Microsoft AD mới:**

1.  Truy cập bảng điều khiển Directory Service.
2.  Chọn **Set up directory** và chọn **AWS Managed Microsoft AD**.
3.  Chọn **Standard Edition** và nhập tên Directory DNS cùng password.
4.  Chọn một virtual private cloud (VPC), trong ví dụ này sử dụng **Default VPC**.
5.  Chọn **Create directory**.

## Directory administration EC2 instance

Directory administration EC2 instance này sẽ được sử dụng để control Microsoft Active Directory bằng AWS Systems Manager.

**Để deploy directory administration EC2 instance:**

1.  Nếu bạn đã triển khai một directory mới, bạn có thể cần đợi 20–45 phút cho đến khi directory status là **Active**.
2.  Chọn **Directory ID**.
3.  Chọn **Actions** và chọn **Launch directory Administration EC2 Instance**, sử dụng các default options.

Ngoài ra, bạn có thể build các Windows EC2 instances của riêng mình với một role có policy *AmazonSSMManagedInstanceCore*, join nó vào Active Directory domain, và install Active Directory management tools.

**Để remotely connect đến directory administration EC2 instance:**

1.  Truy cập bảng điều khiển **Systems Manager**.
2.  Mở **Fleet Manager** từ navigation pane.
3.  Chọn **Node ID** cho instance có tên kết thúc bằng **managementInstance**.
4.  Chọn **Node Actions** (phía trên bên phải), chọn **Connect**, và sau đó chọn **Connect with Remote Desktop**.
5.  Nhập username `admin` và directory password mà bạn đã đặt trước đó.

### Create a test Active Directory user

Bạn sẽ sử dụng test user account này để sign in vào một EC2 instance và initiate một command mô phỏng suspicious activity dẫn đến account này bị disabled.

**Để sử dụng directory administration EC2 instance để tạo một test user trong Active Directory:**

1.  Từ **management EC2 instance**, mở start menu, chọn **Windows Administrative Tools** và sau đó mở **Active Directory Users and Computers**.
2.  Browse đến **Domain** của bạn, **Domain OU**, và sau đó đến **Users OU**, nhấp chuột phải và chọn **New** và sau đó chọn **User**.
3.  Tạo một **TestUser** user, đảm bảo rằng bạn không chọn **Account is disabled**.

### Create a privileged domain service account

Bạn sẽ tạo domain user account này với delegated permissions để được sử dụng bởi Systems Manager Windows Service.

**Để sử dụng directory administration EC2 instance để tạo một service account trong AD:**

1.  Từ **management EC2 instance**, mở start menu, chọn **Windows Administrative Tools**, và sau đó mở **Active Directory Users and Computers**.
2.  Browse đến **Domain** của bạn, **Domain OU**, và sau đó đến **Users OU**. Nhấp chuột phải và chọn **New**, và sau đó chọn **User**.
3.  Tạo một **SSMService** user, đảm bảo rằng bạn không chọn **Account is disabled**.

**Để ủy quyền cho service account trong AD:**

1.  Nhấp chuột phải vào **Users OU** và chọn **Delegate Control**.
2.  Chọn **Next** trên **Delegation of Control Wizard**.
3.  **Add** service user mới mà bạn đã tạo trước đó và chọn **Next**.
4.  Chọn **Create a custom task to delegate** và chọn **Next**.
5.  Chọn **Only the following objects in the folder** và chọn **User Objects**, sau đó chọn **Next**.
6.  Chọn **General** và **Property-specific** để hiển thị các permissions, chọn **Read userAccountControl** và **Write userAccountControl** (gần cuối list), sau đó chọn **Next** và **Finish**.

**Để thêm một service account vào local administrators group:**

1.  Từ **management EC2 instance**, mở start menu, chọn **Windows Administrative Tools**, và sau đó mở **Computer Management**.
2.  Browse đến **Local Users and Groups**, sau đó đến **Groups**.
3.  Nhấp chuột phải vào **Administrators** và chọn **Properties**.
4.  Chọn **Add** để thêm service user mới mà bạn đã tạo trước đó và chọn **OK**.

### Cấu hình Systems Manager

Cấu hình Systems Manager trên directory administration EC2 instance với permission để manage Active Directory.

**Để cấu hình Systems Manager:**

1.  Từ **management EC2 instance** từ **Start Menu**, chọn **Windows Administrative Tools**, và sau đó mở **Services**.
2.  Tìm **Amazon SSM Agent**, nhấp chuột phải, và chọn **Properties**.
3.  Chọn tab **Log On** và chọn **This account**.
4.  Trong phần **This account**, nhập **privileged domain username** bạn đã tạo trước đó, theo sau là @ và sau đó là domain name, ví dụ *SSMService@corp.example.com*. Nhập password của bạn và chọn **OK**.

    ![Figure 3: Microsoft Windows Services showing Systems Manager Agent settings](/images/Figure-3-1.png)

    Hình 3: Microsoft Windows Services hiển thị cài đặt Systems Manager Agent

5.  Chọn **OK** trên cửa sổ popups **This account has been granted Log On As A Service right** và **The new logon name will not take effect until you stop and restart the service**.
6.  Nhấp chuột phải vào **Amazon SSM Agent** và chọn **Restart**.

## Systems Manager Run Command

Run Command là một feature của Systems Manager có thể remote và securely manage cấu hình của managed nodes của bạn. Bạn có thể sử dụng Run Command để automate các administrative tasks phổ biến và perform one-time configuration changes at scale. Bạn có thể sử dụng Run Command từ console, [AWS Command Line Interface (AWS CLI)](https://aws.amazon.com/cli), [AWS Tools for PowerShell](https://aws.amazon.com/powershell/), hoặc AWS SDKs. Run Command được cung cấp at no additional cost.

**Để tạo một Run Command document bằng PowerShell command để vô hiệu hóa domain user accounts:**

1.  Truy cập bảng điều khiển AWS Systems Manager.
2.  Chọn **Documents** trong phần **Change Management Tools**.
3.  Chọn **Create document** và chọn **Command or Session**.
4.  Nhập tên, ví dụ *DisableADUser*.
5.  Chọn document type **Command**.
6.  Chọn **YAML** và sau đó nhập mã sau:

    ```yaml
    ---
    schemaVersion: "2.2"
    description: "Disable AD Users"
    parameters:
      UserName:
        type: String
        description: "(Required) The username to disable."
    mainSteps:
    - action: "aws:runPowerShellScript"
      name: "DisableUser"
      inputs:
        runCommand:
        - "import-module activedirectory"
        - "$disableuser = get-aduser {{ UserName }} | select-object -ExpandProperty DistinguishedName"
        - "dsmod user $disableuser -disabled yes"

    ```

7.  Chọn Create document.

**Để tạo một Run Command document bằng bash command để tìm username từ UserID:**

1.  Follow steps 1–3 từ procedure trước.
2.  Nhập tên, ví dụ GetUsernameFromID.
3.  Chọn document type Command.
4.  Chọn YAML và sau đó nhập mã sau:

    ```yaml
    ---
    description: "Get Username from Linux"
    schemaVersion: "2.2"
    parameters:
      UserId:
        type: String
        description: "(Required) The User ID to find."
        default: "1000"
    mainSteps:
    - action: aws:runShellScript
      name: GetLinuxUsername
      precondition:
        StringEquals:
        - platformType
        - Linux
      inputs:
        timeoutSeconds: 7200
        runCommand:
          - "#!/bin/bash"
          - "#"
          - "UserName=$(id -nu {{ UserId }})"
          - "if [[ $UserName == *'@'* ]]; then"
          - "echo ${UserName%@*} "
          - "else if [[ $UserName == *'\\'* ]]; then"
          - "echo $UserName | sed 's/.*\\\\//g'"
          - "fi"
          - "fi"
      outputs:
        - Name: output
          Selector: $.Payload.output
          Type: String
    ```

5.  Chọn Create document.

## Step Functions

Step Functions là một serverless orchestration service mà bạn có thể sử dụng để coordinate nhiều AWS services, microservices và third-party integrations vào business-critical applications. Step Functions được sử dụng rộng rãi để orchestrate complex workflows, chẳng hạn như loan processing, fraud detection, risk management, và compliance processes. Bằng cách breaking down các processes này thành một series of steps, Step Functions cung cấp một overview rõ ràng và control của toàn bộ workflow. Điều này giúp make sure rằng nó executes mỗi stage correctly và in the right order. Một trong những critical aspects của việc sử dụng Step Functions trong regulated industries là importance of security và data protection.

By the end of this section, state machine của bạn sẽ có một sequential flow bắt đầu bằng một choice mặc định là "No UserID found" và với "UserID" hiện diện, bao gồm các steps *Find Username*, *Wait*, *Get Username*, và *Disable AD User*. Nếu không, bạn có thể drag actions vào đúng order hoặc change the next state associated with mỗi action. Alternatively, copy [state machine definition JSON](https://aws-security-blog-content.s3.us-east-1.amazonaws.com/public/sample/2686-how-to-automatically-disable-users-in-aws-managed-microsoft-ad-based-on-guardduty-findings/Step-function-definition.json) này và import trực tiếp vào Step Functions.

**Để tạo một Step Functions state machine để thực thi Systems Manager Run Commands:**

1.  Truy cập bảng điều khiển Step Functions.
2.  Chọn **Get Started**.
3.  Chọn **Create your own**.
4.  Nhập tên cho state machine, chọn **Standard**, và chọn **Continue**.
5.  Chọn **JSONPath** làm state machine query language.
6.  Từ navigation pane, search for và add action **Pass** bằng cách **dragging the action** vào center window.
7.  Thêm Action **Systems Manager: SendCommand** để Finding the Username using Run Command.
8.  Chọn **SendCommand**, change the state name thành Find Username, và sau đó nhập mã sau vào **API Parameters** ở phía bên phải màn hình.

    ```yaml
    {
      "DocumentName": "GetUsernameFromID",
      "Parameters": {
        "UserId.$": "States.Array(States.JsonToString($.detail.service.runtimeDetails.process.euid))"
      },
      "Targets": [
        {
          "Key": "InstanceIds",
          "Values.$": "States.Array($.detail.resource.instanceDetails.instanceId)"
        }
      ]
    }
    ```

9.  Với **SendCommand** được chọn, chọn tab **Input/Output**, chọn **Add original input to output using ResultPath**, chọn **Combine original input with result**, và nhập như sau:

    ```
    $.RunCommand.State
    ```

10. Thêm action **Wait** và set **number of seconds** để đợi trước khi resuming the execution là 5 giây.
11. Thêm action **Systems Manager**: **GetCommandInvocation**, action này sẽ get Username value từ Run Command và change the state name thành **Get Username**, sau đó nhập các API Parameters sau.

    ```
    {
      "CommandId.$": "$.RunCommand.State.Command.CommandId",
      "InstanceId.$": "$.detail.resource.instanceDetails.instanceId"
    }
    ```

12. Trên tab **Input/Output**, chọn **Transform result with ResultSelector** và nhập như sau:

    ```
    {
      "StandardOutputContent.$": "States.StringSplit($.StandardOutputContent,'\n')"
    }
    ```

13. Thêm action **Systems Manager: SendCommand** sẽ disable Active Directory user using Run Command. Change the state name thành **Disable AD User** sau đó nhập các API Parameters sau, changing the **InstanceIds** value thành ID của Active Directory Management server của bạn.

    ```
    {
      "DocumentName": "DisableADUser",
      "Parameters": {
        "UserName.$": "$.StandardOutputContent"
      },
      "Targets": [
        {
          "Key": "InstanceIds",
          "Values": [
            "i-0b22a22eec53b9321"
          ]
        }
      ]
    }
    ```

14. Thêm action **Choice**, chọn **pencil icon** next to Rule #1, chọn **Edit conditions**, nhập variable *$.detail.service.runtimeDetails.process.euid*, chọn operator **is present**, value **true**, để **Not** là blank, và chọn **Save Conditions**.
15. Re-arrange the state machine layout to the same structure as displayed in Figure 4, với một sequential flow bắt đầu bằng một choice mặc định là "No UserID found" và với "UserID" hiện diện bao gồm các steps **Find Username**, **Wait**, **Get Username**, và **Disable AD User**.

    ![Figure 4: Step Functions state machine structure](/images/Figure-4-1.png)

    Hình 4: Cấu trúc Step Functions state machine

16. Chọn **Create** (top right) và sau đó **Confirm** để tạo step function state machine.

**To add permissions to enable the State Machine to run System Manager commands:**

1.  Trong newly created state machine, chọn **Config** (top center).
2.  Chọn **View in IAM**, under **Permissions**, **Execution role**.
3.  Chọn **Add permissions**, **Attach Polices** (center right).
4.  Search for và select **AmazonSSMAutomationRole** và chọn **Add permission**.

## EventBridge

EventBridge helps developers build event-driven architectures (EDA) by connecting loosely coupled publishers và consumers using event routing, filtering, và transformation. To create an EventBridge rule that triggers the Systems Manger Run Command document bạn đã tạo trước đó:

1.  Truy cập bảng điều khiển Amazon EventBridge và chọn Create rule with EventBridge Rule.
2.  Nhập tên, ví dụ GuardDutyDisableADuser.
3.  Chọn Rule with an event pattern và chọn Next.
4.  Trong cửa sổ Event pattern JSON, chọn Edit pattern và nhập mã sau:

    ```
    {
      "source": ["aws.guardduty"],
      "detail-type": ["GuardDuty Finding"]
    }
    ```

5.  Chọn **Next**.
6.  Chọn **AWS Service**.
7.  Chọn **Step Functions state machine** làm target.
8.  Chọn state machine bạn đã tạo trước đó, ví dụ **MyStateMachine-A123456789**.
9.  Chọn **Next** hai lần và chọn **Create rule**.

## Tạo một test EC2 instance

Để generate alerts trên GuardDuty, bạn tạo một domain joined Linux EC2 instance. Trong ví dụ này, bạn sẽ sử dụng hai separate EC2 instances để có thể monitor for activity từ mỗi instance trong GuardDuty và sử dụng EventBridge để create automations.

**To create an [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) role to permit the EC2 instance to join the AD:**

1.  Truy cập bảng điều khiển IAM.
2.  Chọn **Policies** từ navigation pane.
3.  Chọn **Create policy** (top right).
4.  Chọn Policy editor **JSON**, nhập mã sau và chọn **Next**.

    ```
    {
    "Version": "2012-10-17",
    "Statement": [
    	{
    		"Effect": "Allow",
    		"Action": [
    			"secretsmanager:GetSecretValue",
    			"secretsmanager:DescribeSecret"
    			],
    		"Resource": "*"
    	}
    	]
    }
    ```

5.  Nhập **Policy name**, ví dụ *SecretsManagerGetSecrets*, và chọn **Create policy**.
6.  Chọn **Roles** từ navigation pane.
7.  Chọn **Create role** (top right).
8.  Chọn **AWS service** và chọn **EC2** từ service or use case selection, sau đó chọn **Next**.
9.  Search for và select các policies sau và chọn **Next**.
    - AmazonSSMDirectoryServiceAccess
    - AmazonSSMManagedInstanceCore
    - SecretsManagerGetSecrets (được tạo trước đó)
10. Nhập **role name**, ví dụ *EC2DomainJoin*, và chọn **Create role**.

**Để tạo một secret sẽ được sử dụng để lưu trữ thông tin xác thực đặc quyền được sử dụng để join EC2 instances vào domain:**

1.  Truy cập bảng điều khiển Secrets Manager.
2.  Chọn **Store a new secret**.
3.  Chọn **Other type of secret**.
4.  Add các keys sau với corresponding value của domain username và password có permissions để join computers to the domain:
    1.  awsSeamlessDomainUsername
    2.  awsSeamlessDomainPassword
5.  Chọn **Next**.
6.  **Nhập secret name sau**, thay thế `<d-1234567890>` bằng directory ID của bạn.

    ```
    aws/directory-services/<d-1234567890>/seamless-domain-join
    ```

7.  Chọn **Next** hai lần, sau đó **Store**.

For more information, see [Seamlessly joining an Amazon EC2 Linux instance to your AWS Managed Microsoft AD Active Directory](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ad_connector_seamlessly_join_linux_instance.html).

**Để tạo một EC2 instance đã tham gia domain để thử nghiệm GuardDuty automation:**

1.  Truy cập bảng điều khiển Amazon EC2.
2.  Chọn **Instances** từ navigation pane.
3.  Chọn **Launch Instances**.
4.  Chọn **Amazon Linux AMI**.
5.  Chọn một existing **Key Pair** hoặc create new key pair.
6.  Scroll đến cuối và chọn **Advanced details**.
7.  Trong **Domain join directory**, chọn domain.
8.  Trong **IAM instance profile**, chọn **EC2DomainJoin** role mà bạn đã tạo trước đó.
9.  Chọn **Launch Instance**.

## Kiểm thử

Để simulate một threat, hãy sử dụng một GuardDuty test domain mà GuardDuty sẽ recognize là một command and control server.

1.  Truy cập bảng điều khiển Amazon EC2.
2.  Chọn **Instances** từ navigation pane.
3.  Chọn test EC2 instance mà bạn đã tạo trước đó.
4.  Chọn **Connect**, chọn tab **Session Manager**, và chọn **Connect**.
5.  Authenticate với test user của bạn bằng cách nhập `su` theo sau là test user với domain name mà bạn đã tạo trước đó. Ví dụ `su TestUser@example.com`, sau đó nhập password.
6.  Nhập lệnh `curl guarddutyc2activityb.com`.
    - Bạn sẽ receive an error vì page sẽ không resolve được, nhưng GuardDuty sẽ have detected suspicious events.
7.  Truy cập bảng điều khiển GuardDuty và chọn **Findings** từ navigation pane.
8.  Trong vòng 3–5 phút, bạn sẽ thấy một high severity finding cho **Backdoor:EC2/C&CActivity.B!DNS**.

> **Lưu ý**: Bạn phải archive GuardDuty finding trước khi re-running test này, vì EventBridge rule chỉ runs once against a GuardDuty finding with the same details. Để archive the finding, chọn check box next to the **Backdoor:EC2/C&CActivity.B!DNS** finding, chọn **Actions** (top right), và chọn **Archive**.
>

![Figure 5: GuardDuty simulated findings](/images/Figure-5-1.png)

Hình 5: Các simulated findings của GuardDuty

Nếu bạn quay lại **Active Directory Users and Computers** trên Directory Administration EC2 instance, bạn sẽ thấy rằng Test User hiện đã bị disabled. Bạn có thể enable user bằng cách nhấp chuột phải vào user và chọn **Enable Account**.

![Figure 6: Active Directory Users and Computers showing the disabled test use](/images/Figure-6-1.png)

Hình 6: Active Directory Users and Computers hiển thị test user bị disabled

## Kết luận

Trong bài đăng này, bạn đã học cách triển khai AWS Managed AD, Systems Manager Run Command, EventBridge, Step Functions và GuardDuty để monitor suspicious events và disable the associated Active Directory user account.

Bạn có thể expand scenario này bằng cách tạo các Run Command documents đặt lại Active Directory passwords, disable computer accounts, hoặc Active Directory tasks được hỗ trợ bởi Microsoft PowerShell. Ngoài ra, bạn có thể add steps trong Step Functions state machine để notify administrators thông qua [Amazon Simple Notification Service (Amazon SNS)](https://aws.amazon.com/sns) hoặc add additional checks với [AWS Lambda](https://aws.amazon.com/lambda).

Mặc dù bài đăng này sử dụng AWS Managed Microsoft AD, functionality tương tự có thể achieved với việc manual deployment của Active Directory trên Amazon EC2 hoặc on-premises, either by using an EC2 instance joined to the Active Directory domain with the Active Directory administration tools installed hoặc bằng cách installing Systems Manager agent onto a management server on-premises.

Nếu bạn có feedback về bài đăng này, hãy submit **comments** trong phần Comments bên dưới. Nếu bạn có questions về bài đăng này, hãy start a new thread trên [AWS re:Post GuardDuty](https://repost.aws.com/tags/TAkQ_AMw65SICuEGEmuUXv4g/amazon-guardduty) hoặc [contact AWS Support](https://console.aws.amazon.com/support/home).

![Tim-Kingdon.jpg](/images/Tim-Kingdon.jpg)

**Tim Kingdon**

Tim là Senior Partner Solutions Architect tại AWS và có hơn 25 năm kinh nghiệm trong các ngành healthcare, financial, government, và defence industries. Trong role của mình, ông cung cấp strategic technical guidance cho partners và helps drive success của họ thông qua technical enablement initiatives.

**TAGS:** [Amazon GuardDuty](https://aws.amazon.com/blogs/security/tag/amazon-guardduty/), [GuardDuty](https://aws.amazon.com/blogs/security/tag/guardduty/), [Security Blog](https://aws.amazon.com/blogs/security/tag/security-blog/)

---

## 📖 Glossary - Thuật ngữ

| English | Tiếng Việt | Định nghĩa |
|---------|------------|------------|
| Auto Scaling | Tự động mở rộng quy mô | Khả năng tự động tăng/giảm resources dựa trên demand |
| Load Balancer | Bộ cân bằng tải | Phân phối traffic đến multiple servers |
| Microservices | Kiến trúc microservices | Architectural pattern chia application thành small services |
| Amazon GuardDuty | Amazon GuardDuty | Dịch vụ phát hiện mối đe dọa tự động của AWS |
| AWS Directory Service for Microsoft Active Directory | Dịch vụ Thư mục AWS cho Microsoft Active Directory | Dịch vụ quản lý Active Directory trên AWS |
| Amazon EC2 (Elastic Compute Cloud) | Amazon EC2 | Dịch vụ tính toán đám mây cung cấp các máy ảo |
| Amazon S3 (Simple Storage Service) | Amazon S3 | Dịch vụ lưu trữ đối tượng có khả năng mở rộng cao |
| Amazon EventBridge | Amazon EventBridge | Dịch vụ bus sự kiện không máy chủ để kết nối ứng dụng |
| AWS Step Functions | AWS Step Functions | Dịch vụ điều phối quy trình làm việc không máy chủ |
| AWS Systems Manager | AWS Systems Manager | Bộ công cụ quản lý vận hành đám mây |
| Run Command | Run Command | Tính năng của Systems Manager để thực thi lệnh từ xa |
| AWS CLI (Command Line Interface) | AWS CLI | Công cụ dòng lệnh để quản lý các dịch vụ AWS |
| AWS Tools for PowerShell | Công cụ AWS cho PowerShell | Công cụ dựa trên PowerShell để quản lý các dịch vụ AWS |
| AWS SDKs (Software Development Kits) | AWS SDKs | Bộ công cụ phát triển phần mềm cho các ngôn ngữ lập trình khác nhau |
| AWS IAM (Identity and Access Management) | AWS IAM | Dịch vụ quản lý quyền truy cập và nhận dạng |
| Secrets Manager | Secrets Manager | Dịch vụ quản lý, lưu trữ, và truy xuất bí mật một cách an toàn |
| Session Manager | Session Manager | Tính năng của Systems Manager để quản lý phiên Shell trên các phiên bản |
| Amazon SNS (Simple Notification Service) | Amazon SNS | Dịch vụ nhắn tin cho ứng dụng đến ứng dụng (A2A) và ứng dụng đến người (A2P) |
| AWS Lambda | AWS Lambda | Dịch vụ tính toán không máy chủ để chạy mã mà không cần cấp phép hoặc quản lý máy chủ |
| Domain Controller | Bộ điều khiển miền | Máy chủ quản lý tài nguyên mạng và người dùng trong miền AD |
| Availability Zone | Vùng sẵn sàng | Vị trí tách biệt về mặt vật lý trong một Vùng AWS, được thiết kế để cô lập lỗi |
| VPC (Virtual Private Cloud) | VPC (Đám mây riêng ảo) | Mạng ảo riêng biệt được phân lập logic trong AWS Cloud |
| Instance | Phiên bản | Một máy chủ ảo trong EC2 |
| Policy | Chính sách | Một tài liệu JSON quy định các quyền trong IAM |
| Role | Vai trò | Một thực thể IAM có các quyền được đính kèm |
| Key Pair | Cặp khóa | Một bộ khóa công khai và riêng tư được sử dụng để bảo mật truy cập vào các phiên bản EC2 |
| Runtime Monitoring | Giám sát thời gian chạy | Tính năng của GuardDuty giám sát hoạt động của quy trình và tệp trong các phiên bản EC2 |
| Finding | Phát hiện | Một cảnh báo bảo mật được tạo bởi GuardDuty |
| State Machine | Máy trạng thái | Một quy trình công việc được định nghĩa trong AWS Step Functions |
| Event Pattern | Mẫu sự kiện | Một bộ lọc được sử dụng trong EventBridge để khớp các sự kiện |
| Target | Mục tiêu | Một dịch vụ AWS được EventBridge kích hoạt khi một sự kiện khớp với mẫu |
| Service Account | Tài khoản dịch vụ | Một tài khoản người dùng được thiết kế để chạy các ứng dụng hoặc dịch vụ |
| Delegation of Control | Ủy quyền kiểm soát | Quyền được trao cho người dùng hoặc nhóm để quản lý các đối tượng trong Active Directory |

## 🔗 Tài liệu tham khảo

### Tài liệu gốc
- [Bài viết gốc](https://aws.amazon.com/blogs/security/how-to-automatically-disable-users-in-aws-managed-microsoft-ad-based-on-guardduty-findings/)

### Tài liệu tiếng Việt
- [AWS Learning Resources](https://aws.amazon.com/vi/training/): Tài nguyên học tập AWS
- [Community Discussions](https://repost.aws/): Thảo luận cộng đồng AWS (re:Post)

### Tools và Services
- [Amazon GuardDuty](https://aws.amazon.com/guardduty/): Dịch vụ phát hiện mối đe dọa tự động của AWS
- [AWS Directory Service for Microsoft Active Directory](https://aws.amazon.com/directoryservice/): Dịch vụ quản lý Active Directory trên AWS
- [Amazon EC2](https://aws.amazon.com/ec2/): Dịch vụ máy chủ ảo trong đám mây
- [Amazon EventBridge](https://aws.amazon.com/eventbridge/): Dịch vụ bus sự kiện không máy chủ
- [AWS Step Functions](https://aws.amazon.com/step-functions/): Dịch vụ điều phối quy trình làm việc không máy chủ
- [AWS Systems Manager](https://aws.amazon.com/systems-manager/): Nền tảng hoạt động hợp nhất cho tài nguyên AWS
- [Amazon S3](https://aws.amazon.com/s3/): Dịch vụ lưu trữ đối tượng
- [AWS CLI](https://aws.amazon.com/cli/): Giao diện dòng lệnh của AWS
- [AWS Tools for PowerShell](https://aws.amazon.com/powershell/): Bộ công cụ AWS cho PowerShell
- [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/): Dịch vụ quản lý danh tính và truy cập
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/): Dịch vụ quản lý bí mật
- [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html): Tính năng của Systems Manager để quản lý phiên Shell trên các phiên bản
- [Amazon SNS](https://aws.amazon.com/sns/): Dịch vụ thông báo đơn giản
- [AWS Lambda](https://aws.amazon.com/lambda/): Dịch vụ tính toán không máy chủ

---

## 💬 Ghi chú của người dịch

Bài dịch này được thực hiện dựa trên khả năng hiểu và xử lý ngôn ngữ tự nhiên của mô hình AI.

### Challenges trong quá trình dịch
-   **Technical Terms**: Đảm bảo độ chính xác của các thuật ngữ kỹ thuật chuyên ngành AWS và Active Directory khi giữ nguyên bằng tiếng Anh trong ngữ cảnh tiếng Việt. Việc lựa chọn từ ngữ tiếng Việt phù hợp nhất để vừa truyền tải đúng nghĩa vừa giữ được tính chuyên nghiệp là một thách thức.
-   **Complex Concepts**: Giải thích các khái niệm phức tạp về kiến trúc tự động hóa GuardDuty, EventBridge, Step Functions và Systems Manager một cách rõ ràng, dễ hiểu đối với độc giả Việt Nam, trong khi vẫn duy trì các thuật ngữ gốc.
-   **Consistency**: Duy trì sự nhất quán trong cách sử dụng thuật ngữ tiếng Anh và cách dịch các phần văn bản tiếng Việt còn lại xuyên suốt toàn bộ bài viết, đặc biệt với các dịch vụ và tính năng của AWS.

### Insights gained
-   **Technical Learning**: Nâng cao hiểu biết về cách các dịch vụ bảo mật và tự động hóa của AWS có thể được tích hợp để tạo ra các giải pháp phản ứng mối đe dọa hiệu quả, đặc biệt là trong môi trường Active Directory.
-   **Language Skills**: Cải thiện khả năng diễn đạt các khái niệm kỹ thuật phức tạp trong tiếng Việt một cách tự nhiên và chính xác, đồng thời vẫn giữ nguyên các thuật ngữ gốc.
-   **Industry Knowledge**: Có cái nhìn sâu hơn về các thực tiễn tốt nhất trong việc tự động hóa phản ứng sự cố bảo mật trong môi trường đám mây lai.

---

## 🤝 Đóng góp và Feedback

Bài dịch này được thực hiện trong khuôn khổ **FCJ Internship Program**.

**📧 Liên hệ**: truongdinhvukhanh13@gmail.com

**💬 Feedback**: Mọi góp ý để cải thiện chất lượng dịch thuật xin gửi về email trên

**🔄 Updates**: Bài dịch sẽ được cập nhật dựa trên feedback từ cộng đồng

---

*© 2025 - Bản dịch thuộc về Trương Đình Vũ Khanh. Vui lòng credit khi sử dụng.*
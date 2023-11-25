---
layout: page
title: "Chương 03: Hàm"
permalink: /chapter-03/
nav_order: 4
---
# **CHƯƠNG 03: HÀM**

---

Trong những buổi đầu của việc lập trình, chúng tôi soạn thảo các hệ thống câu lệnh và các chương trình con. Sau đó, trong thời đại của Fortran và PL/1, chúng tôi soạn thảo các hệ thống chương trình, chương trình con, và các hàm. Ngày nay, chỉ còn các hàm là tồn tại. Các hàm là những viên gạch xây dựng nên chương trình. Và chương này sẽ giúp bạn tạo nên những viên gạch chắc chắn cho chương trình của bạn.

Hãy xem xét code trong Listing 3-1. Thật khó để tìm thấy một hàm dài. Nhưng sau một lúc tìm kiếm, tôi đã thấy nó. Nó không chỉ dài, mà còn có code trùng lặp, nhiều thứ dư thừa, các kiểu dữ liệu và API lạ,... Xem bạn phải sử dụng bao nhiêu giác quan trong ba phút tới để hiểu được nó:

**\# Listing 3-1: HtmlUtil.java (FitNesse 20070619)**

```java
public static String testableHtml(PageData pageData, boolean includeSuiteSetup) 
throws Exception {
    WikiPage wikiPage = pageData.getWikiPage();
    StringBuffer buffer = new StringBuffer();
    if (pageData.hasAttribute("Test")) {
        if (includeSuiteSetup) {
            WikiPage suiteSetup =
                    PageCrawlerImpl.getInheritedPage(
                            SuiteResponder.SUITE_SETUP_NAME, wikiPage);
            if (suiteSetup != null) {
                WikiPagePath pagePath =
                    suiteSetup.getPageCrawler().getFullPath(suiteSetup);
                String pagePathName = PathParser.render(pagePath);
                buffer.append("!include -setup .")
                    .append(pagePathName)
                    .append("\n");
            }
        }
        WikiPage setup =
            PageCrawlerImpl.getInheritedPage("SetUp", wikiPage);
        if (setup != null) {
            WikiPagePath setupPath =
                wikiPage.getPageCrawler().getFullPath(setup);
            String setupPathName = PathParser.render(setupPath);
            buffer.append("!include -setup .")
                .append(setupPathName)
                .append("\n");
        }
    }
    buffer.append(pageData.getContent());
    if (pageData.hasAttribute("Test")) {
        WikiPage teardown =
        PageCrawlerImpl.getInheritedPage("TearDown", wikiPage);
        if (teardown != null) {
            WikiPagePath tearDownPath =
                wikiPage.getPageCrawler().getFullPath(teardown);
            String tearDownPathName = PathParser.render(tearDownPath);
            buffer.append("\n")
                .append("!include -teardown .")
                .append(tearDownPathName)
                .append("\n");
        }
        if (includeSuiteSetup) {
            WikiPage suiteTeardown =
                PageCrawlerImpl.getInheritedPage(
                SuiteResponder.SUITE_TEARDOWN_NAME,
                wikiPage);
            if (suiteTeardown != null) {
                WikiPagePath pagePath =
                    suiteTeardown.getPageCrawler()
                    .getFullPath(suiteTeardown);
                String pagePathName = PathParser.render(pagePath);
                buffer.append("!include -teardown .")
                    .append(pagePathName)
                    .append("\n");
            }
        }
    }
    pageData.setContent(buffer.toString());
    return pageData.getHtml();
}
```

Bạn có hiểu hàm trên sau ba phút đọc không? Chắc chắn là không. Có quá nhiều thứ xảy ra với nhiều mức độ trừu tượng khác nhau. Các chuỗi kỳ lạ, các lời gọi hàm trộn lẫn cùng các câu lệnh if lồng nhau,...

Tuy nhiên, chỉ với một vài phép rút gọn đơn giản, đặt lại vài cái tên, và một chút tái cơ cấu lại hàm, tôi đã có thể nắm bắt được mục đích của hàm này trong chín dòng lệnh. Thử sức lại với nó trong ba phút tiếp theo nào:

**\# Listing 3-2: HtmlUtil.java (refactored)**

```java
public static String renderPageWithSetupsAndTeardowns(PageData pageData, boolean isSuite)
throws Exception {
    boolean isTestPage = pageData.hasAttribute("Test");
    if (isTestPage) {
        WikiPage testPage = pageData.getWikiPage();
        StringBuffer newPageContent = new StringBuffer();
        includeSetupPages(testPage, newPageContent, isSuite);
        newPageContent.append(pageData.getContent());
        includeTeardownPages(testPage, newPageContent, isSuite);
        pageData.setContent(newPageContent.toString());
    }
    return pageData.getHtml();
}
```

Trừ khi bạn là học viên của FitNesse, nếu không bạn sẽ không hiểu đầy đủ hàm này. Nhưng không sao, bạn có thể hiểu rằng hàm này thực hiện một số việc thiết lập và chia nhỏ trang, sao đó hiển thị chúng dưới dạng HTML. Nếu đã quen với JUnit, bạn có thể nhận ra hàm này thuộc về một framework nào đó. Và, dĩ nhiên, những gì tôi vừa nói là hoàn toàn chính xác. Việc dự đoán chức năng của hàm từ Listing 3-2 khá dễ dàng, nhưng trong Listing 3-1 điều đó gần như là không thể.

Vậy điều gì làm cho hàm trong Listing 3-2 dễ đọc và dễ hiểu? Bằng cách nào chúng ta có thể tạo nên một hàm thể hiện được chức năng của nó? Những đặc tính nào cho phép một người đọc bình thường hiểu được chương trình mà họ đang cùng làm việc?

## Nhỏ!!!

Nguyên tắc đầu tiên của hàm là chúng phải nhỏ. Nguyên tắc thứ hai là chúng phải nhỏ hơn nữa. Đây không phải là một khẳng định mà tôi có thể chứng minh. Tôi không thể cung cấp bất kỳ tài liệu hay nghiên cứu nào khẳng định rằng hàm nhỏ là tốt hơn. Những gì tôi có thể nói với bạn là trong gần bốn thập kỷ, tôi đã viết các hàm với nhiều kích cỡ khác nhau. Tôi đã viết 3000 dòng lệnh ghê tởm, tôi đã viết các hàm trong phạm vi từ 100 đến 300 dòng, và tôi đã viết các hàm dài từ 20 đến 30 dòng. Kinh nghiệm đã dạy tôi một điều quý giá rằng, các hàm nên rất nhỏ.

Vào những năm 80, chúng tôi cho rằng một hàm không nên lớn hơn một màn hình. Dĩ nhiên, chúng tôi nói điều đó khi các màn hình VT100 chỉ có 24 dòng cùng 80 cột, và 4 dòng đầu thì được dùng cho mục đích quản trị. Ngày nay, với một phông chữ thích hợp và một màn hình xịn, bạn có thể phủ đến 150 ký tự cho 100 dòng hoặc nhiều hơn trên một màn hình. Các dòng code không nên dài quá 150 ký tự. Các hàm không nên "chạm nóc" 100 dòng, và độ dài thích hợp nhất dành cho hàm là không quá 20 dòng lệnh.

Vậy thu gọn một hàm bằng cách nào? Năm 1999 tôi có đến thăm Kent Beck tại nhà của ông ở Oregon. Chúng tôi ngồi xuống và cùng nhau viết một số chương trình nhỏ. Ông ấy đã cho tôi xem một chương trình nhỏ được viết bằng Java/Swing mà ông ấy gọi là Sparkle (Tia Sáng). Nó tạo ra một hiệu ứng hình ảnh trên màn hình rất giống với cây đũa thần của các bà tiên đỡ đầu. Khi bạn di chuyển chuột, các tia sáng lấp lánh sẽ "nhỏ giọt" từ con trỏ chuột xuống đáy cửa sổ, cứ như bị lực hấp dẫn kéo xuống vậy. Khi Kent cho tôi xem mã nguồn, tôi đã bị ấn tượng bởi độ nhỏ gọn của các hàm [...]. Mọi hàm trong chương trình này chỉ dài hai, ba hoặc bốn dòng. Mỗi hàm đều rõ ràng. Mỗi hàm kể một câu chuyện. Và mỗi hàm dẫn bạn đến hàm tiếp theo hấp dẫn hơn. Đó là cách hàm của bạn trở nên ngắn gọn.

Hàm của bạn sẽ ngắn như thế nào? Chúng thường phải ngắn hơn Listing 3-2! Thật vậy, Listing 3-2 thực sự nên được rút gọn thành Listing 3-3.

**\# Listing 3-3: HtmlUtil.java (re-refactored)**

```java
public static String renderPageWithSetupsAndTeardowns(PageData pageData, boolean isSuite)
throws Exception {
    if (isTestPage(pageData))
        includeSetupAndTeardownPages(pageData, isSuite);
    return pageData.getHtml();
}
```

### Các khối lệnh và thụt dòng

Điều này có nghĩa là các khối lệnh bên trong câu lệnh `if`, `else`, `while`,... phải dài một dòng. Và dòng đó nên là một lời gọi hàm. Điều này không chỉ giữ các hàm kèm theo nhỏ mà còn bổ sung thêm giá trị tài liệu cho code của bạn, vì các hàm được gọi có thể có một cái tên thể hiện được mục đích của nó.

Điều này cũng có nghĩa các hàm không nên được thiết kế lớn để chứa các cấu trúc lồng nhau. Do đó, lời gọi hàm không nên thụt lề quá mức hai. Điều này, dĩ nhiên là làm cho các hàm dễ đọc và dễ viết hơn.

## Thực hiện MỘT việc

Rõ ràng là Listing 3-1 đang làm nhiều hơn một việc. Nó tạo bộ đệm, tìm nạp trang, tìm kiếm các trang được kế thừa, hiển thị đường dẫn, thêm chuỗi phức tạp và tạo HTML,... Listing 3-1 bận rộn làm nhiều việc khác nhau. Mặt khác, Listing 3-3 làm một việc đơn giản. Nó tạo các thiết lập và hiển thị nội dung vào các trang thử nghiệm.

Lời khuyên dưới đây đã xuất hiện nhiều lần, dưới dạng này hoặc dạng khác trong hơn 30 năm qua:

> "_HÀM CHỈ NÊN THỰC HIỆN MỘT VIỆC. CHÚNG NÊN LÀM TỐT VIỆC ĐÓ,
 VÀ CHỈ LÀM DUY NHẤT VIỆC ĐÓ"_

Vấn đề là, chúng ta khó biết "một việc" ở đây là việc gì. Listing 3-3 có làm một việc không? Thật dễ để chỉ ra nó đang làm 3 việc:

1. Xác định đây có phải là trang thử nghiệm hay không
2. Nếu phải, nạp vào các cài đặt và tái thiết lập nó
3. Hiển thị trang bằng HTML

Vậy, cái gì đây? Hàm đang thực hiện một việc hay ba việc? [...] Chúng ta có thể mô tả hàm bằng cách xem nó như một đoạn TO ngắn (Ngôn ngữ LOGO sử dụng từ khóa TO giống như cách Ruby và Python sử dụng def. Vì vậy, mọi hàm đều bắt đầu bằng từ TO. Điều này tạo nên một hiệu ứng thú vị trên các hàm được thiết kế):

TO RenderPageWithSetupsAndTeardowns (ĐỂ hiển thị trang với các cài đặt và tái nạp), chúng tôi kiểm tra xem trang có phải là trang thử nghiệm hay không và nếu có, chúng tôi sẽ đưa vào các cài đặt và tái thiết lập nó. Sau đó, chúng tôi sẽ hiển thị trang bằng HTML.

Nếu hàm thực hiện các chức năng thấp hơn tên của hàm, thì hàm đó vẫn đang làm một việc. Sau tất cả, lý do chúng tôi viết các hàm là để phân tích một khái niệm lớn thành các khái niệm nhỏ hơn (nói cách khác, là phân tích tên hàm thành các tên ở mức độ thấp hơn).

Rõ ràng là Listing 3-1 gồm nhiều chức năng với nhiều mức độ khác nhau, và hiển nhiên là nó đang làm nhiều hơn một việc. Ngay cả Listing 3-2 cũng có hai mức độ, và đã được chứng minh bằng cách thu gọn nó. Nhưng sẽ rất khó để rút gọn Listing 3-3. Chúng ta có thể trích xuất câu lệnh if thành một hàm có tên `includeSetupsAndTeardownsIfTestPage`, nhưng điều đó chỉ đơn giản là mang code đến nơi khác mà không thay đổi mức độ trừu tượng của nó.

Vì vậy, một cách khác để biết hàm đang làm nhiều hơn "một việc" là khi bạn có thể trích xuất một hàm khác từ nó, nhưng với một cái tên khác so với chức năng của nó ở trong hàm.

[...]

## Mỗi hàm là một cấp độ trừu tượng

Để đảm bảo các hàm của chúng ta đang thực hiện "một việc", chúng ta cần chắc chắn rằng các câu lệnh trong hàm của chúng ta đều ở cùng cấp độ trừu tượng. Hãy xem cách Listing 3-1 vi phạm quy tắc này. Có những khái niệm trong đó có mức trừu tượng rất cao, chẳng hạn như `getHtml();` những thứ khác ở mức trừu tượng trung gian, chẳng hạn như: `String pagePathName = PathParser.render (pagePath)` và những người khác có mức độ thấp đáng kể, chẳng hạn như: `.append("\n")`.

Việc trộn lẫn các cấp độ trừu tượng với nhau trong một hàm sẽ luôn gây ra những hiểu lầm cho người đọc. [...]

### Đọc code từ trên xuống dưới: Nguyên tắc Stepdown

Chúng tôi muốn code được đọc tuần tự từ trên xuống. Chúng tôi muốn mọi hàm được theo sau bởi các hàm có cấp độ trừu tượng lớn hơn để chúng tôi có thể đọc chương trình. Và khi chúng tôi xem xét một danh sách các khai báo hàm, mức độ trừu tượng của chúng phải được giảm dần. Tôi gọi đó là nguyên tắc Stepdown (tạm dịch: nguyên tắc ruộng bậc thang).

Nói cách khác, chúng tôi muốn đọc chương trình như thể đọc một bài văn có nhiều đoạn. Mỗi phần mô tả một cấp độ trừu tượng hiện tại và liên kết tới các đoạn văn tiếp theo, với cấp độ trừu tượng tiếp theo.

```
_To include the setups and teardowns, we include setups, then we include the test page content, and then we include the teardowns._

_To include the setups, we include the suite setup if this is a suite, then we include the regular setup._

_To include the suite setup, we search the parent hierarchy for the "SuiteSetUp" page and add an include statement with the path of that page._

_To search the parent. . ._
```

Sự thật là rất khó để các lập trình viên học cách tuân theo nguyên tắc này và viết các hàm ở một mức độ trừu tượng duy nhất. Nhưng học thủ thuật này cũng rất quan trọng. Nó là chìa khóa để đảm bảo các hàm ngắn gọn và giữ cho các chúng làm "một việc". Làm cho code của bạn đọc như một đoạn văn là kỹ thuật hiệu quả để duy trì sự đồng nhất của các cấp trừu tượng.

[...]

## Câu lệnh switch

Thật khó để tạo nên một câu lệnh `switch` nhỏ (và cả chuỗi lệnh `if/else`). Ngay cả câu lệnh `switch` chỉ có 2 trường hợp. Và cũng rất khó để tạo ra một câu lệnh `switch` mà chỉ làm "một việc". Bởi bản chất của chúng, các câu lệnh `switch` luôn thực hiện N việc. Rất tiếc là, không phải lúc nào chúng tôi cũng tránh được chúng, nhưng chúng tôi có thể đảm bảo rằng các câu lệnh `switch` được chôn giấu trong một lớp cơ sở và không bao giờ được lặp lại. Chúng tôi làm việc này, dĩ nhiên, bằng tính chất đa hình.

Xem xét Listing 3-4 dưới đây. Nó hiển thị hoạt động dựa vào loại nhân viên:

**\# Listing 3-4: Payroll.java**

```java
public Money calculatePay(Employee e)
throws InvalidEmployeeType {
    switch (e.type) {
        case COMMISSIONED:
            return calculateCommissionedPay(e);
        case HOURLY:
            return calculateHourlyPay(e);
        case SALARIED:
            return calculateSalariedPay(e);
        default:
            throw new InvalidEmployeeType(e.type);
    }
}
```

Có một số vấn đề với hàm này. Đầu tiên, nó lớn, và khi có loại nhân viên mới được thêm vào, nó sẽ to ra. Thứ hai, rất rõ ràng, nó đang làm nhiều hơn "một việc". Thứ ba, nó vi phạm Nguyên tắc Đơn nhiệm (Single Responsibility Principle – SRP) vì có nhiều lý do để nó thay đổi. Thứ tư, nó vi phạm Nguyên tắc Đóng & mở (Open Closed Principle – OCP) vì nó phải thay đổi khi có loại nhân viên khác được thêm vào. Nhưng vấn đề tồi tệ nhất của hàm này là có vô hạn các hàm khác có cùng cấu trúc. Ví dụ, chúng ta có thể có:

```java
isPayday(Employee e, Date date),
```

hoặc

```java
deliverPay(Employee e, Money pay),
```

hoặc một loạt những hàm khác. Tất cả đều có cấu trúc giống nhau!

> _Nguyên tắc Đơn nhiệm: Mỗi lớp chỉ nên chịu trách nhiệm về một nhiệm vụ cụ thể nào đó mà thôi._
_Nguyên tắc Đóng & mở: Chúng ta nên hạn chế việc chỉnh sửa bên trong một Class hoặc Module có sẵn, thay vào đó hãy xem xét mở rộng chúng.

Giải pháp cho vấn đề này (xem Listing 3-5) là chôn câu lệnh `switch` trong một lớp cơ sở của ABSTRACT FACTORY (tìm hiểu ABSTRACT FACTORY tại: [https://vi.wikipedia.org/wiki/Abstract_factory](https://vi.wikipedia.org/wiki/Abstract_factory)), và không bao giờ để người khác trông thấy nó. ABSTRACT FACTORY sẽ sử dụng câu lệnh switch để tạo ra các trường hợp thích hợp của các dẫn xuất của `Employee`,và các hàm khác như `calculatePay`, `isPayday`, và `deliverPay`, sẽ được gọi bằng tính đa hình thông qua _interface_ của `Employee`.

**\# Listing 3-5: Employee and Factory**

```java
public abstract class Employee {
    public abstract boolean isPayday();
    public abstract Money calculatePay();
    public abstract void deliverPay(Money pay);
}
/*...*/
public interface EmployeeFactory {
    public Employee makeEmployee(EmployeeRecord r) throws InvalidEmployeeType;
}
/*...*/
public class EmployeeFactoryImpl implements EmployeeFactory {
    public Employee makeEmployee(EmployeeRecord r) throws 	InvalidEmployeeType {
        switch (r.type) {
            case COMMISSIONED:
                return new CommissionedEmployee(r) ;
            case HOURLY:
                return new HourlyEmployee(r);
            case SALARIED:
                return new SalariedEmploye(r);
            default:
                throw new InvalidEmployeeType(r.type);
        }
    }
}
```

Nguyên tắc chung của tôi dành cho các câu lệnh switch là chúng có thể được tha thứ nếu chúng chỉ xuất hiện một lần, được sử dụng để tạo các đối tượng đa hình, và được ẩn đằng sau bằng tính kế thừa để phần còn lại của hệ thống không nhìn thấy chúng. Tất nhiên, nguyên tắc cũng chỉ là nguyên tắc, và trong nhiều trường hợp tôi đã vi phạm một hoặc nhiều phần của nguyên tắc đó.

## Dùng tên có tính mô tả

Trong Listing 3-7, tôi đã thay đổi tên hàm ví dụ từ `testableHtml` thành `SetupTeardownIncluder.render`. Đây là một tên tốt hơn nhiều vì nó mô tả được chức năng của hàm đó. Tôi cũng đã cung cấp cho từng phương thức riêng (private method) một tên mô tả như `isTestable` hoặc `includeSetupAndTeardownPages`. Thật khó để xem thường giá trị của những cái tên tốt. Hãy nhớ đến nguyên tắc của Ward: _"Bạn biết bạn đang làm việc cùng code sạch là khi việc đọc code hóa ra yomost hơn những gì bạn mong đợi"._ Một nửa chặn đường để đạt được nguyên tắc đó là chọn được một cái tên "xịn" cho những hàm làm "một việc". Hàm càng nhỏ và càng cô đặc thì càng dễ chọn tên mô tả cho nó hơn.

Đừng ngại đặt tên dài. Một tên dài nhưng mô tả đầy đủ chức năng của hàm luôn tốt hơn những cái tên ngắn. Và một tên dài thì tốt hơn một ghi chú (comment) dài. Dùng một nguyên tắc đặt tên cho phép dễ đọc nhiều từ trong tên hàm, và những từ đó sẽ cho bạn biết hàm đó hoạt động ra sao.

Đừng ngại dành thời gian cho việc chọn tên. Thật vậy, bạn nên thử một số tên khác nhau và đọc lại code ngay sau đó. Các IDE hiện đại như Eclipse hay IntelliJ làm cho việc đổi tên trở nên dễ dàng hơn rất nhiều. Sử dụng một trong những IDE đó và thử đặt các tên khác nhau cho đến khi bạn tìm thấy một cái tên có tính mô tả.

Chọn một cái tên có tính mô tả tốt sẽ giúp bạn vẽ lại thiết kế của mô-đun đó vào não, và việc cải thiện nó sẽ đơn giản hơn. Nhưng điều đó không có nghĩa là bạn sẽ bất chấp tất cả để "săn" được một cái tên tốt hơn để thay thế tên hiện tại.

[...]

## Đối số của hàm

Số lượng đối số lý tưởng cho một hàm là không (niladic), tiếp đến là một (monadic), sau đó là hai (dyadic). Nên tránh trường hợp ba đối số (triadic) nếu có thể. Các hàm có nhiều hơn ba đối số (polyadic) chỉ cần thiết trong các trường hợp đặc biệt, và sau đó nên hạn chế sử dụng chúng đến mức thấp nhất.

Các đối số có những vấn đề của nó. Nó làm mất nhiều khái niệm của chương trình khi xuất hiện. Đó là lý do tại sao tôi đã loại bỏ gần như tất cả chúng ở ví dụ trên. Hãy xem xét `StringBuffer` trong ví dụ: Chúng tôi có thể đưa nó vào lời gọi hàm để tạo đối số thay vì để nó làm một biến thể hiện thông thường, nhưng sau đó độc giả của chúng ta sẽ phải "tự thông não" mỗi khi họ nhìn thấy nó. Khi bạn đọc một câu chuyện được viết nên bởi mô-đun, `includeSetupPage()` sẽ dễ hiểu hơn `includeSetupPageInto(newPageContent)`. Đối số có mức trừu tượng khác tên hàm và buộc bạn phải để tâm đến nó, mặc dù nó không quá quan trọng ở thời điểm đó.

[...]

Các đối số đầu ra khó hiểu hơn các đối số đầu vào. Khi chúng ta đọc một hàm, chúng ta quen với ý tưởng thông tin đi vào hàm thông qua các đối số, và kết quả nhận được thông qua giá trị trả về. Chúng tôi thường không nghỉ rằng thông tin trả về được truyền qua các đối số. Vì vậy, các đối số đầu ra thường khiến chúng tôi bất ngờ.

Hàm có một đối số đầu vào sẽ là tốt nhì (tốt nhất vẫn là hàm không có đối số). `SetupTeardownIncluder.render(pageData)` khá dễ hiểu. Rõ ràng là chúng ta sẽ kết xuất (render) dữ liệu của đối tượng `pageData`.

### Hình thức chung của hàm một đối số (monadic)

Có hai lý do phổ biến để bạn truyền một đối số vào hàm. Bạn có thể đặt một câu hỏi đúng - sai cho đối số đó, như boolean `fileExists("MyFile")`. Hoặc bạn có thể thao tác trên đối số đó, biến nó thành một thứ gì khác và trả lại nó. Ví dụ, `InputStream fileOpen("MyFile")` biến đổi một chuỗi tên tệp thành một giá trị `InputStream`. Người đọc thường chỉ mong đợi hai cách này khi nhìn vào hàm có một đối số. Bạn nên chọn tên hàm thấy được sự phân biệt rõ ràng và luôn sử dụng hai hình thức này trong cùng một ngữ cảnh.

Một dạng ít phổ biến hơn nhưng vẫn rất hữu ít dành cho hàm có một đối số, đó là các sự kiện (event). Các hàm dạng này có một đối số đầu vào nhưng không có đối số đầu ra. Toàn bộ chương trình được hiểu là để thông dịch các lời gọi hàm như một sự kiện, và sử dụng các đối số để thay đổi trạng thái của hệ thống, ví dụ, `voidpasswordAttemptFailedNtimes(int attempts)`. Hãy cẩn thận với các hàm kiểu này, nó phải cực rõ ràng để người đọc biết đây là một sự kiện, nhớ chọn tên và ngữ cảnh một cách cẩn thận.

Cố gắng tránh bất kỳ hàm một đối số nào không tuân theo các mẫu trên, ví dụ: `void includeSetupPageInto(StringBuffer pageText)`. Sử dụng một đối số đầu ra thay vì một giá trị trả về khá là khó hiểu. Nếu một hàm chuyển đổi đối số đầu vào của nó, kết quả của phép biến đổi nên xuất hiện dưới dạng giá trị trả về. Thật vậy, `StringBuffer transform(StringBuffer in)` là tốt hơn khi so với `void transform-(StringBuffer out)`. [...]

### Đối số luận lý

"Việc chuyển một đối số boolean vào hàm là một cái gì đó rất khủng khiếp. Nó ngay lập tức chỉ ra hàm của bạn đang là nhiều hơn một việc. Một việc nó làm khi đối số đúng, và một việc được làm khi đối số sai". Tuy nhiên, không phải lúc nào việc này cũng tởm lợm như bạn nghĩ. Ở một số trường hợp, việc này là hoàn toàn bình thường.

### Hàm có hai đối số (dyadic)

Hàm có hai đối số sẽ khó hiểu hơn hàm có một đối số. Ví dụ `writeField(name)` sẽ dễ hiểu hơn `writeField(output-Stream, name)`. Mặc dù ý nghĩa của cả hai đều như nhau, đều dễ hiểu khi lần đầu nhìn vào. Nhưng hàm thứ hai yêu cầu bạn phải dừng lại, cho đến khi bạn học được cách bỏ qua tham số đầu tiên. Và, dĩ nhiên, có một vấn đề khi bạn bỏ qua đoạn code nào đó, thì khả năng đoạn code đó chứa lỗi là rất cao.

Tất nhiên luôn có những lúc hai đối số sẽ hợp lý hơn một đối số. Ví dụ: `Point p = new Point(0,0);` là hoàn toàn hợp lý khi bạn đang code về tọa độ mặt phẳng. Chúng tôi sẽ cảm thấy bối rối khi thấy `new Point(0);` trong trường hợp này.

Ngay cả hàm dyadic rõ ràng như `assertEquals(expected, actual)` vẫn có vấn đề. Đã bao nhiêu lần bạn nhầm lẫn vị trí giữa `expected` và `actual`? Hai đối số không có thứ tự tự nhiên. Thứ tự `expected`, `actual` là một quy ước đòi hỏi bạn phải nhớ nó trong đầu.

Những hàm dyadic không phải là những con quỷ dữ, và chắc chắn bạn phải viết chúng. Tuy nhiên bạn nên lưu ý rằng bạn sẽ phải trả giá cho việc đó, và nên tận dụng tối đa những thủ thuật hay lợi thế có sẵn để chuyển chúng về thành dạng monadic. Ví dụ, bạn có thể làm cho phương thức `writeField` trở thành một thành viên của `outputStream` để bạn có thể dùng lệnh `outputStream.writeField(name)`.

### Hàm ba đối số (triadic)

Hàm có ba đối số khó hiểu hơn nhiều so với hàm hai đối số. Các vấn đề về sắp xếp, tạm ngừng và bỏ qua tăng gấp đôi. Tôi đề nghị bạn cẩn thận trước khi tạo ra nó.

[...]

### Đối số đối tượng

Khi một hàm có vẻ cần nhiều hơn hai hoặc ba đối số, có khả năng một số đối số đó phải được bao bọc thành một lớp riêng của chúng. Ví dụ, hãy xem xét sự khác biệt giữa hai khai báo sau đây:

```java
Circle makeCircle(double x, double y, double radius);
Circle makeCircle(Point center, double radius);
```

Giảm số lượng các đối số bằng cách tạo ra các đối tượng có vẻ như gian lận, nhưng không phải. Khi các nhóm biến được chuyển đổi cùng nhau, như cách x và y ở ví dụ trên, chúng có khả năng là một phần của một khái niệm xứng đáng có tên riêng.

**Danh sách đối số**

Đôi khi, chúng tôi muốn chuyển một số lượng đối số vào một hàm. Hãy xem xét ví dụ về phương thức `String.format`:

```java
String.format("%s worked %.2f hours.", name, hours);
```

Nếu tất cả các đối số được xử lý giống nhau, như ví dụ trên, thì tất cả chúng tương đương với một đối số kiểu List. Bởi lý do đó, String.format thực chất là một hàm có hai đối số. Thật vậy, việc khai báo String.format như ví dụ dưới đây rõ ràng là một hàm dyadic:

```java
void monad(Integer... args);
void dyad(String name, Integer... args);
void triad(String name, int count, Integer... args);
```

### Động từ và các từ khóa

Chọn tên tốt cho một hàm có thể góp phần giải thích ý định của hàm và mục đích của các đối số. Trong trường hợp hàm monadic, hàm và đối số nên tạo thành một cặp động từ/danh từ hợp lý. `write(name)` là một ví dụ hoàn hảo trong trường hợp này. Dù cái tên này là gì, nó cho chúng ta biết nó được viết. Một cái tên tốt hơn có lẽ là `writeField(name)`, nó cho chúng ta biết rằng tên là một trường.

Cuối cùng là một ví dụ về dạng từ khóa của tên hàm. Bằng cách này, chúng tôi mã hóa tên của các đối số thành tên hàm. Ví dụ, `assertEquals` có thể được cải tiến thành `assertExpectedEqualsActual(expected, actual)`. Điều này làm giảm vấn đề về việc nhớ vị trí của các đối số.

## Không có tác dụng phụ

Tác dụng phụ (hay hiệu ứng lề) là một sự lừa dối. Hàm của bạn được hy vọng sẽ làm một việc, nhưng nó cũng làm những việc khác mà bạn không thấy. Đôi khi nó bất ngờ làm thay đổi giá trị biến của lớp của nó. Hoặc nó sẽ biến chúng thành các tham số được truyền vào hàm, hoặc các hàm toàn cục. Trong cả hai trường hợp, chúng tạo ra các sai lầm và làm sai kết quả.

Hãy xem xét hàm trong ví dụ dưới đây. Hàm này sử dụng thuật toán để kiểm tra `userName` và `password`. Nó trả về true nếu chúng khớp và trả về false nếu có gì sai. Nhưng nó cũng có tác dụng phụ. Ban phát hiện ra nó chứ?

**\# Listing 3-6: UserValidator.java**

```java
public class UserValidator {
    private Cryptographer cryptographer;
    public boolean checkPassword(String userName, String password) {
        User user = UserGateway.findByName(userName);
        if (user != User.NULL) {
            String codedPhrase = user.getPhraseEncodedByPassword();
            String phrase = cryptographer.decrypt(codedPhrase, password);
            if ("Valid Password".equals(phrase)) {
                Session.initialize();
                return true;
            }
        }
        return false;
    }
}
```

Tác dụng phụ ở đây là lời gọi hàm `Session.initialize()`. Hàm checkPassword, theo cách đặt tên của nó, nói rằng nó chỉ kiểm tra mật khẩu. Tên hàm không thông báo rằng nó khởi tạo session (Tìm hiểu thêm: [https://en.wikipedia.org/wiki/Session_(computer_science)](https://en.wikipedia.org/wiki/Session_(computer_science))). Vậy nên khi ai đó dùng hàm này, họ tin rằng mình chỉ kiểm tra tính hợp lệ của người dùng mà không biết dữ liệu của session hiện tại có nguy cơ bị mất.

Tác dụng phụ này tạo ra một mắt xích về thời gian. Đó là, `checkPassword` chỉ được gọi vào những thời điểm nhất định (nói cách khác, khi nó an toàn để khởi tạo session). Nếu được gọi lung tung, dữ liệu của session có thể vô tình bị mất. Mắt xích tạm thời này gây khó hiểu, đặc biệt là nó lúc ẩn lúc hiện. Nếu bạn có một mắt xích như vậy, bạn nên làm nó hiện rõ trong tên hàm. Trong trường hợp này, chúng ta có thể đổi tên hàm thành `checkPasswordAndInitializeSession`, mặc dù chắc chắn hàm này vi phạm nguyên tắc "Làm một việc".

### Đối số đầu ra

[...]

Nói chung chúng ta nên tránh các đối số đầu ra. Nếu hàm của bạn phải thay đổi trạng thái của một cái gì đó, hãy thay đổi trạng thái của đối tượng sở hữu nó.

## Tách lệnh truy vấn

Hàm nên làm một cái gì đó hoặc trả lời một cái gì đó, nhưng không phải cả hai. Hoặc là hàm của bạn thay đổi trạng thái của một đối tượng, hoặc nó sẽ trả về một số thông tin về đối tượng đó. Làm cả hai thường gây nên sự n    hầm lẫn. Xem xét hàm ví dụ sau:

```java
publicboolean set(String attribute, String value);
```

Hàm này đặt giá trị cho thuộc tính nếu thuộc tính đó tồn tại. Nó trả về true nếu thành công, và false nếu thất bại. Điều này dẫn đến các câu lệnh lẻ như sau:

```java
if (set("username", "unclebob"))...
```

Hãy tưởng tượng điều này từ quan điểm của người đọc. Nó có nghĩa là gì? Nó hỏi thuộc tính "`username`" đã được đặt thành "`unclebob`" chưa? Hay nó hỏi thuộc tính "`username`" trước đó có giá trị là "`unclebob`"? Thật khó để suy ra ý nghĩa của hàm vì không rõ từ "set" là động từ hay tính từ.

Dự định của tác giả là đặt set trở thành một động từ, nhưng trong ngữ cảnh của câu lệnh `if`, nó mang đến cảm giác như một tính từ [...]. Chúng tôi thử giải quyết vấn đề này bằng cách đổi tên hàm đã đặt thành `setAndCheckIfExists`, nhưng điều đó không giúp ích gì nhiều trong ngữ cảnh của câu lệnh if. Giải pháp thực sự là tách lệnh khỏi truy vấn sao cho sự nhầm lẫn không thể xảy ra.

```java
if (attributeExists("username")) {
    setAttribute("username", "unclebob");
    ...
}
```

[...]

## Prefer Exceptions to Returning Error Codes

[...]

## Đừng lặp lại code của bạn

Xem xét lại Listing 3-1 một cách cẩn thận, bạn sẽ nhận thấy rằng có một thuật toán được lặp lại bốn lần. Mỗi lần cho mỗi trường hợp `SetUp`, `SuiteSetUp`, `TearDown`, và `SuiteTearDown`. Không dễ dàng để phát hiện ra sự trùng lặp này vì cả bốn trường hợp code được trộn lẫn với code khác, và sự sao chép là không thống nhất. Tuy nhiên, việc trùng lặp code như vậy là một vấn đề, vì nó làm code của bạn phình to ra và khi cần sửa đổi, bạn sẽ phải sửa đổi bốn lần. Điều đó cũng đồng nghĩa với việc nguy cơ xuất hiện lỗi là bốn lần.

Vấn đề này được khắc phục bằng phương thức `include` trong Listing 3-7. Đọc lại code một lần nữa và bạn sẽ thấy khả năng đọc của toàn bộ mô-đun được tăng lên chỉ bằng cách giảm sự trùng lặp đó.

Sự trùng lặp có lẽ là gốc rễ của mọi tội lỗi trong lập trình. Nhiều nguyên tắc và kinh nghiệm đã được tạo ra cho mục đích kiểm soát hoặc loại bỏ nó. Lập trình cấu trúc, lập trình hướng đối tượng (OOP), lập trình hướng khía cạnh (Aspect Oriented Programming – AOP), lập trình hướng thành phần (Component Oriented Programming – COP), tất cả chúng đều có chiến lược để loại bỏ code trùng lặp. Nó chứng minh rằng kể từ khi chương trình con được phát minh, các sáng kiến trong ngành công nghiệp phát triển phần mềm đều nhắm đến việc loại bỏ những đoạn code trùng lặp ra khỏi mã nguồn.

## Lập trình có cấu trúc

Một số lập trình viên đi theo nguyên tắc lập trình có cấu trúc của Edsger Dijkstra. Dijkstra nói rằng mọi hàm, và mọi khối trong hàm nên có một lối vào và một lối thoát. Điều đó có nghĩa là chỉ nên có một lệnh `return` trong hàm, không có câu lệnh `break`, `continue` trong một vòng lặp; và không bao giờ dùng bất kỳ câu lệnh goto nào.

Chúng tôi thông cảm với các nguyên tắc và mục tiêu của lập trình cấu trúc, nhưng các nguyên tắc này chỉ mang lại một chút lợi ích khi các hàm bạn viết rất nhỏ. Ở các hàm lớn hơn, lợi ích mà nó mang lại thật sự là không đáng kể.

Vậy nên nếu bạn có thể tiếp tục giữ cho các hàm của mình nhỏ, thì việc sử dụng các câu lệnh `return`, `break` hay `continue` là vô hại và đôi khi nó còn giúp hàm của bạn rõ ràng hơn nguyên tắc một lối vào, một lối thoát. Mặt khác, lệnh `goto` chỉ có ý nghĩa trong các hàm lớn, vì vậy nên tránh sử dụng nó.

## Tôi đã viết các hàm này như thế nào?

Viết phần mềm cũng giống như viết các thể loại khác. Khi bạn viết một bài báo hay một văn kiện, bạn sẽ suy nghĩ trước, sau đó bạn nhào nặn nó cho đến khi nó trở nên mạch lạc, trơn tru. Các bản thảo ban đầu có thể vụng về và rời rạc, vì vậy bạn vứt nó vào sọt rác và tái cơ cấu nó, tinh chỉnh nó cho đến khi nó được đọc theo cách mà bạn muốn.

Khi tôi bắt đầu viết các hàm, chúng dài và phức tạp. Chúng có rất nhiều vòng lặp lồng nhau, chúng có hàng tá đối số. Các tên được đặt tùy ý, và tồn tại nhiều code trùng lặp. Nhưng tôi cũng có một bộ unit test để đảm bảo cho tất cả những dòng code vụng về đó.

Và sau đó, tôi thay đổi và tinh chỉnh lại code đó, tách ra thành các hàm, đặt lại tên và loại bỏ code trùng lặp. Tôi thu nhỏ phương thức và sắp xếp lại chúng. Đôi khi tôi _đập tan nát_ một lớp, trong khi vẫn giữ lại các bài test đã hoàn thành.

Cuối cùng, các hàm tôi hoàn thành đã tuân theo các nguyên tắc tôi đặt ra trong chương này. Tôi không tuân theo các nguyên tắc tôi đặt ra để bắt đầu viết nó, điều đó là không thể.

## Kết luận

Mỗi hệ thống được xây dựng từ một DSL được thiết kế và mô tả bởi các lập trình viên. Các hàm là một động từ, và các lớp là một danh từ [...]. Nghệ thuật lập trình, dĩ nhiên, luôn là nghệ thuật sử dụng ngôn ngữ.

Các lập trình viên tài năng xem các hệ thống như những câu chuyện kể, chứ không phải là các chương trình được viết. Họ sử dụng khả năng của ngôn ngữ lập trình mà họ chọn để diễn đạt _câu chuyện_ phong phú hơn và giàu cảm xúc hơn. Một phần của các DSL là cấu trúc phân cấp của các hàm mô tả hành động diễn ra trong hệ thống đó. Và các hàm được định nghĩa để nói lên câu chuyện của riêng mình.

Chương này đã chỉ cho bạn về cách viết tốt các hàm. Nếu bạn tuân thủ các nguyên tắc trên, các hàm của bạn sẽ ngắn gọn, được đặt tên và được tổ chức tốt. Nhưng đừng bao giờ quên rằng mục tiêu của bạn là kể một câu chuyện về hệ thống, và các hàm bạn viết cần ăn khớp với nhau một cách rõ ràng và chính xác để giúp bạn hoàn thành việc đó.

## SetupTeardownIncluder

**\# Listing 3-7: SetupTeardownIncluder.java**

```java
package fitnesse.html;
import fitnesse.responders.run.SuiteResponder;
import fitnesse.wiki.*;
public class SetupTeardownIncluder {
    private PageData pageData;
    private boolean isSuite;
    private WikiPage testPage;
    private StringBuffer newPageContent;
    private PageCrawler pageCrawler;

    public static String render(PageData pageData) throws Exception {
        return render(pageData, false);
    }

    public static String render(PageData pageData, boolean isSuite)
    throws Exception {
        return new SetupTeardownIncluder(pageData).render(isSuite);
    }

    private SetupTeardownIncluder(PageData pageData) {
        this.pageData = pageData;
        testPage = pageData.getWikiPage();
        pageCrawler = testPage.getPageCrawler();
        newPageContent = new StringBuffer();
    }

    private String render(boolean isSuite) throws Exception {
        this.isSuite = isSuite;
        if (isTestPage())
            includeSetupAndTeardownPages();
        return pageData.getHtml();
    }

    private boolean isTestPage() throws Exception {
        return pageData.hasAttribute("Test");
    }

    private void includeSetupAndTeardownPages() throws Exception {
        includeSetupPages();
        includePageContent();
        includeTeardownPages();
        updatePageContent();
    }

    private void includeSetupPages() throws Exception {
        if (isSuite)
            includeSuiteSetupPage();
        includeSetupPage();
    }

    private void includeSuiteSetupPage() throws Exception {
        include(SuiteResponder.SUITE_SETUP_NAME, "-setup");
    }

    private void includeSetupPage() throws Exception {
        include("SetUp", "-setup");
    }

    private void includePageContent() throws Exception {
        newPageContent.append(pageData.getContent());
    }

    private void includeTeardownPages() throws Exception {
        includeTeardownPage();
        if (isSuite)
            includeSuiteTeardownPage();
    }

    private void includeTeardownPage() throws Exception {
        include("TearDown", "-teardown");
    }

    private void includeSuiteTeardownPage() throws Exception {
        include(SuiteResponder.SUITE_TEARDOWN_NAME, "-teardown");
    }

    private void updatePageContent() throws Exception {
        pageData.setContent(newPageContent.toString());
    }

    private void include(String pageName, String arg) throws Exception {
        WikiPage inheritedPage = findInheritedPage(pageName);
        if (inheritedPage != null) {
            String pagePathName = getPathNameForPage(inheritedPage);
            buildIncludeDirective(pagePathName, arg);
        }
    }

    private WikiPage findInheritedPage(String pageName) throws Exception {
        return PageCrawlerImpl.getInheritedPage(pageName, testPage);
    }

    private String getPathNameForPage(WikiPage page) throws Exception {
        WikiPagePath pagePath = pageCrawler.getFullPath(page);
        return PathParser.render(pagePath);
    }

    private void buildIncludeDirective(String pagePathName, String arg) {
        newPageContent
        .append("\n!include ")
        .append(arg)
        .append(" .")
        .append(pagePathName)
        .append("\n");
    }
}
```

## Tham khảo

**[KP78]:** Kernighan and Plaugher, _The Elements of Programming Style_, 2d. ed., McGrawHill, 1978.

**[PPP02]:** Robert C. Martin, _Agile Software Development: Principles, Patterns, and Practices_, Prentice Hall, 2002.

**[GOF]:**_Design Patterns: Elements of Reusable Object Oriented Software_, Gamma et al., Addison-Wesley, 1996.

**[PRAG]:**_The Pragmatic Programmer_, Andrew Hunt, Dave Thomas, Addison-Wesley, 2000.

**[SP72]:**_Structured Programming_, O.-J. Dahl, E. W. Dijkstra, C. A. R. Hoare, Academic Press, London, 1972.
_- Viết bởi Tim Ottinger_

---

## Giới thiệu

Những cái tên có ở khắp mọi nơi trong phần mềm. Chúng ta đặt tên cho các biến, các hàm, các đối số, các lớp và các gói của chúng ta. Chúng ta đặt tên cho những file mã nguồn và thư mục chứa chúng. Chúng ta đặt tên cho những file _\*.jar_, file _\*.war,.._. Chúng ta đặt tên và đặt tên. Vì chúng ta đặt tên rất nhiều, nên chúng ta cần làm tốt điều đó. Sau đây là một số quy tắc đơn giản để tạo nên những cái tên tốt.

## Dùng những tên thể hiện được mục đích

Điều này rất dễ. Nhưng chúng tôi muốn nhấn mạnh rằng chúng tôi nghiêm túc trong việc này. Chọn một cái tên "xịn" mất khá nhiều thời gian, nhưng lại tiết kiệm (thời gian) hơn sau đó. Vì vậy, hãy quan tâm đến cái tên mà bạn chọn và chỉ thay đổi chúng khi bạn sáng tạo ra tên "xịn" hơn. Những người đọc code của bạn (kể cả bạn) sẽ _sung sướng_ hơn khi bạn làm điều đó.

Tên của biến, hàm, hoặc lớp phải trả lời tất cả những câu hỏi về nó. Nó phải cho bạn biết lý do nó tồn tại, nó làm được những gì, và dùng nó ra sao. Nếu có một comment đi kèm theo tên, thì tên đó không thể hiện được mục đích của nó.

```java
int d; // elapsed time in days
```

Tên **d** không tiết lộ điều gì cả. Nó không gợi lên cảm giác gì về thời gian, cũng không liên quan gì đến ngày. Chúng ta nên chọn một tên thể hiện được những gì đang được cân đo, và cả đơn vị đo của chúng:

```java
int elapsedTimeInDays;
int daysSinceCreation;
int daysSinceModification;
int fileAgeInDays;
```

Việc chọn tên thể hiện được mục đích có thể làm cho việc hiểu và thay đổi code dễ dàng hơn nhiều. Hãy đoán xem mục đích của đoạn code dưới đây là gì?

```java
public List<int[]> getThem() {
    List<int[]> list1 = newArrayList<int[]>();
    for (int[] x : theList)
        if (x[0] == 4)
            list1.add(x);
    return list1;
}
```

Tại sao lại nói khó mà biết được đoạn code này đang làm gì? Không có biểu thức phức tạp, khoảng cách và cách thụt đầu dòng hợp lý, chỉ có 3 biến và 2 hằng số được đề cập. Thậm chí không có các lớp (class) và phương thức đa hình nào, nó chỉ có một danh sách mảng (hoặc thứ gì đó trông giống vậy).

Vấn đề không nằm ở sự đơn giản của code mà nằm ở ý nghĩa của code, do bối cảnh không rõ ràng. Đoạn code trên bắt chúng tôi phải tìm câu trả lời cho các câu hỏi sau:

1. theList chứa cái gì?
2. Ý nghĩa của chỉ số 0 trong phần tử của theList?
3. Số 4 có ý nghĩa gì?
4. Danh sách được return thì dùng kiểu gì?

Câu trả lời không có trong code, nhưng sẽ có ngay sau đây. Giả sử chúng tôi đang làm game _dò mìn_. Chúng tôi thấy rằng giao diện trò chơi là một danh sách các ô vuông (cell) được gọi là theList. Vậy nên, hãy đổi tên nó thành gameBoard.

Mỗi ô trên màn hình được biểu diễn bằng một sanh sách đơn giản. Chúng tôi cũng thấy rằng chỉ số của số 0 là vị trí biểu diễn giá trị trạng thái (status value), và giá trị 4 nghĩa là trạng thái _được gắn cờ (flagged)._ Chỉ bằng cách đưa ra các khái niệm này, chúng tôi có thể cải thiện mã nguồn một cách đáng kể:

```java
public List<int[]> getFlaggedCells() {
    List<int[]> flaggedCells = newArrayList<int[]>();
    for (int[] cell : gameBoard)
        if (cell[STATUS_VALUE] == FLAGGED)
            flaggedCells.add(cell);
    return flaggedCells;
}
```

Cần lưu ý rằng mức độ đơn giản của code vẫn không thay đổi, nó vẫn chính xác về toán tử, hằng số, và các lệnh lồng nhau,...Nhưng đã trở nên rõ ràng hơn rất nhiều.

Chúng ta có thể đi xa hơn bằng cách viết một lớp đơn giản cho các ô thay vì sử dụng các mảng kiểu int. Nó có thể bao gồm một hàm thể hiện được mục đích (gọi nó là _isFlagged – được gắn cờ_ chẳng hạn) để giấu đi những con số ma thuật _(Từ gốc: magic number – Một khái niệm về các hằng số, tìm hiểu thêm tại_ [https://en.wikipedia.org/wiki/Magic_number_(programming)](https://en.wikipedia.org/wiki/Magic_number_(programming)) _)._

```java
public List<Cell> getFlaggedCells() {
    List<Cell> flaggedCells = newArrayList<Cell>();
    for (Cell cell : gameBoard)
        if (cell.isFlagged())
            flaggedCells.add(cell);
    return flaggedCells;
}
```

Với những thay đổi đơn giản này, không quá khó để hiểu những gì mà đoạn code đang trình bày. Đây chính là sức mạnh của việc chọn tên tốt.

## Tránh sai lệch thông tin

Các lập trình viên phải tránh để lại những dấu hiệu làm code trở nên khó hiểu. Chúng ta nên tránh dùng những từ mang nghĩa khác với nghĩa cố định của nó. Ví dụ, các tên biến như `hp`, `aix` và `sco` là những tên biến vô cùng tồi tệ, chúng là tên của các nền tảng Unix hoặc các biến thể. Ngay cả khi bạn đang code về cạnh huyền (hypotenuse) và hptrông giống như một tên viết tắt tốt, rất có thể đó là một cái tên tồi.

Không nên quy kết rằng một nhóm các tài khoản là một `accountList` nếu nó không thật sự là một danh sách (`List`). Từ _danh sách_ có nghĩa là một thứ gì đó cụ thể cho các lập trình viên. Nếu các tài khoản không thực sự tạo thành danh sách, nó có thể dẫn đến một kết quả sai lầm. Vậy nên, `accountGroup` hoặc `bunchOfAccounts`, hoặc đơn giản chỉ là accounts sẽ tốt hơn.

Cẩn thận với những cái tên gần giống nhau. Mất bao lâu để bạn phân biệt được sự khác nhau giữa `XYZControllerForEfficientHandlingOfStrings` và `XYZControllerForEfficientStorageOfStrings` trong cùng một module, hay đâu đó xa hơn một chút? Những cái tên gần giống nhau như thế này thật sự, thật sự rất khủng khiếp cho lập trình viên.

Một kiểu khủng bố tinh thần khác về những cái tên không rõ ràng là ký tự L viết thường và O viết hoa. Vấn đề? Tất nhiên là nhìn chúng gần như hoàn toàn giống hằng số không và một, kiểu như:

```
int a = l;  
if ( O == l ) a = O1;  
else l = 01;
```

Bạn nghĩ chúng tôi _xạo_? Chúng tôi đã từng khảo sát, và kiểu code như vậy thực sự rất nhiều. Trong một số trường hợp, tác giả của code đề xuất sử dụng phông chữ khác nhau để tách biệt chúng. Một giải pháp khác có thể được sử dụng là truyền đạt bằng lời nói hoặc để lại tài liệu cho các lập trình viên sau này có thể hiểu nó. Vấn đề được giải quyết mà không cần phải đổi tên để tạo ra một sản phẩm khác.

## Tạo nên những sự khác biệt có nghĩa

Các lập trình viên tạo ra vấn đề cho chính họ khi viết code chỉ để đáp ứng cho trình biên dịch hoặc thông dịch. Ví dụ, vì bạn không thể sử dụng cùng một tên để chỉ hai thứ khác nhau trong cùng một khối lệnh hoặc cùng một phạm vi, bạn có thể bị "dụ dỗ" thay đổi tên một cách tùy tiện. Đôi khi điều đó làm bạn cố tình viết sai chính tả, và người nào đó quyết định sửa lỗi chính tả đó, khiến trình biên dịch không có khả năng hiểu nó (cụ thể – tạo ra một biến tên _klass_ chỉ vì tên _class_ đã được dùng cho thứ gì đó).

Mặc dù trình biên dịch có thể làm việc với những tên này, nhưng điều đó không có nghĩa là bạn được phép dùng nó. Nếu tên khác nhau, thì chúng cũng có ý nghĩa khác nhau.

Những tên dạng chuỗi số (a1, a2,... aN) đi ngược lại nguyên tắc đặt tên có mục đích. Mặc dù những tên như vậy không phải là không đúng, nhưng chúng không có thông tin. Chúng không cung cấp manh mối nào về ý định của tác giả. Ví dụ:

```java
public static void copyChars(char a1[], char a2[]) {
    for (int i = 0; i < a1.length; i++) {
        a2[i] = a1[i];
    }
}
```

Hàm này dễ đọc hơn nhiều khi _nguyên nhân_ và _mục đích_ của nó được đặt tên cho các đối số.

Những từ gây nhiễu tạo nên sự khác biệt, nhưng là sự khác biệt vô dụng. Hãy tưởng tượng rằng bạn có một lớp `Product`, nếu bạn có một `ProductInfo` hoặc `ProductData` khác, thì bạn đã thành công trong việc tạo ra các tên khác nhau nhưng về mặt ngữ nghĩa thì chúng là một. `Info` và `Data` là các từ gây nhiễu, giống như `a`, `an` và `the`.

Lưu ý rằng không có gì sai khi sử dụng các tiền tố như `a` và `the` để tạo ra những khác biệt hữu ích. Ví dụ, bạn có thể sử dụng `a` cho tất cả các biến cục bộ và tất cả các đối số của hàm. `a` và `the` sẽ trở thành vô dụng khi bạn quyết định tạo một biến `theZork` vì trước đó bạn đã có một biến mang tên `Zork`.

Những từ gây nhiễu là không cần thiết. Từ `variable` sẽ không bao giờ xuất hiện trong tên biến, từ `table` cũng không nên dùng trong tên bảng. `NameString` sao lại tốt hơn `Name`? `Name` có bao giờ là một số đâu mà lại? Nếu `Name` là một số, nó đã phá vỡ nguyên tắc _Tránh sai lệch thông tin._ Hãy tưởng tượng bạn đang tìm kiếm một lớp có tên `Customer`, và một lớp khác có tên `CustomerObject`. Chúng khác nhau kiểu gì? Cái nào chứa lịch sử thanh toán của khách hàng? Còn cái nào chứa thông tin của khách?

Có một ứng dụng minh họa cho các lỗi trên, chúng tôi đã thay đổi một chút về tên để bảo vệ tác giả. Đây là những thứ chúng tôi thấy trong mã nguồn:

```java
getActiveAccount();
getActiveAccounts();
getActiveAccountInfo();
```

Tôi thắc mắc không biết các lập trình viên trong dự án này phải `getActiveAccount` như thế nào!

Trong trường hợp không có quy ước cụ thể, biến `moneyAmount` không thể phân biệt được với `money`; `customerInfo` không thể phân biệt được với `customer`; `accountData` không thể phân biệt được với `account` và `theMessage` với `message` được xem là một. Hãy phân biệt tên theo cách cung cấp cho người đọc những khác biệt rõ ràng.

## Dùng những tên phát âm được

Con người rất giỏi về từ ngữ. Một phần quan trọng trong bộ não của chúng ta được dành riêng cho các khái niệm về từ. Và các từ, theo định nghĩa, có thể phát âm được. Thật lãng phí khi không sử dụng được bộ não mà chúng ta đã tiến hóa nhằm thích nghi với ngôn ngữ nói. Vậy nên, hãy làm cho những cái tên phát âm được đi nào.

Nếu bạn không thể phát âm nó, thì bạn không thể thảo luận một cách bình thường: "Hey, ở đây chúng ta có _bee cee arr three cee enn tee_, và _pee ess zee kyew int_, thấy chứ?" – Vâng, tôi thấy một thằng thiểu năng. Vấn đề này rất quan trọng vì lập trình cũng là một hoạt động xã hội, chúng ta cần trao đổi với mọi người.

Tôi có biết một công ty dùng tên _genymdhms_ (generation date, year, month, day, hour, minute, and second – phát sinh ngày, tháng, năm, giờ, phút, giây), họ đi xung quanh tôi và "gen why emm dee aich emm ess" (cách phát âm theo tiếng Anh). Tôi có thói quen phát âm như những gì tôi viết, vì vậy tôi bắt đầu nói "gen-yah-muddahims". Sau này nó được gọi bởi một loạt các nhà thiết kế và phân tích, và nghe vẫn có vẻ ngớ ngẫn. Chúng tôi đã từng troll nhau như thế, nó rất thú vị. Nhưng dẫu thế nào đi nữa, chúng tôi đã chấp nhận những cái tên xấu xí. Những lập trình viên mới của công ty tìm hiểu ý nghĩa của các biến, và sau đó họ nói về những từ ngớ ngẫn, thay vì dùng các thuật ngữ tiếng Anh cho thích hợp. Hãy so sánh:

```java
class DtaRcrd102 {
    privateDate genymdhms;
    privateDate modymdhms;
    privatefinalString pszqint = "102";
    /* ... */
};
```

và

```java
class Customer {
    privateDate generationTimestamp;
    privateDate modificationTimestamp;
    privatefinalString recordId = "102";
    /* ... */
};
```

Cuộc trò chuyện giờ đây đã thông minh hơn: "Hey, Mikey, take a look at this record! The generation timestamp is set to tomorrow's date! How can that be?"

## Dùng những tên tìm kiếm được

Các tên một chữ cái và các hằng số luôn có vấn đề, đó là không dễ để tìm chúng trong hàng ngàn dòng code.

Người ta có thể dễ dàng tìm kiếm `MAX_CLASSES_PER_STUDENT`, nhưng số 7 thì lại rắc rối hơn. Các công cụ tìm kiếm có thể mở các tệp, các hằng, hoặc các biểu thức chứa số 7 này, nhưng được sử dụng với các mục đích khác nhau. Thậm chí còn tồi tệ hơn khi hằng số là một số có giá trị lớn và ai đó vô tình thay đổi giá trị của nó, từ đó tạo ra một lỗi mà các lập trình viên không tìm ra được.

Tương tự như vậy, tên e là một sự lựa chọn tồi tệ cho bất kỳ biến nào mà một lập trình viên cần tìm kiếm. Nó là chữ cái phổ biến nhất trong tiếng anh và có khả năng xuất hiện trong mọi đoạn code của chương trình. Về vấn đề này, tên dài thì tốt hơn tên ngắn, và những cái tên tìm kiếm được sẽ tốt hơn một hằng số trơ trọi trong code.

Sở thích cá nhân của tôi là chỉ đặt tên ngắn cho những biến cục bộ bên trong những phương thức ngắn. _Độ dài của tên phải tương ứng với phạm vi hoạt động của nó_. Nếu một biến hoặc hằng số được nhìn thấy và sử dụng ở nhiều vị trí trong phần thân của mã nguồn, bắt buộc phải đặt cho nó một tên dễ tìm kiếm. Ví dụ:

```java
for (int j=0; j<34; j++) {
    s += (t[j]*4)/5;
}
```

và

```java
int realDaysPerIdealDay = 4;
constint WORK_DAYS_PER_WEEK = 5;
int sum = 0;
for (int j=0; j < NUMBER_OF_TASKS; j++) {
    int realTaskDays = taskEstimate[j] * realDaysPerIdealDay;
    int realTaskWeeks = (realdays / WORK_DAYS_PER_WEEK);
    sum += realTaskWeeks;
}
```

Lưu ý rằng sum ở ví dụ trên, dù không phải là một tên đầy đủ nhưng có thể tìm kiếm được. Biến và hằng được cố tình đặt tên dài, nhưng hãy so sánh việc tìm kiếm WORK\_DAYS\_PER\_WEEK dễ hơn bao nhiêu lần so với số 5, đó là chưa kể cần phải lọc lại danh sách tìm kiếm và tìm ra những trường hợp có nghĩa.

## Tránh việc mã hóa

Các cách mã hóa hiện tại là đủ với chúng tôi. Mã hóa các kiểu dữ liệu hoặc phạm vi thông tin vào tên chỉ đơn giản là thêm một gánh nặng cho việc giải mã. Điều đó không hợp lý khi bắt nhân viên phải học thêm một "ngôn ngữ" mã hóa khác khác ngoài các ngôn ngữ mà họ dùng để làm việc với code. Đó là một gánh nặng không cần thiết, các tên mã hóa ít khi được phát âm và dễ bị đánh máy sai.

### Ký pháp Hungary

Ngày trước, khi chúng tôi làm việc với những ngôn ngữ mà độ dài tên là một thách thức, chúng tôi đã loại bỏ sự cần thiết này. Fortran bắt buộc mã hóa bằng những chữ cái đầu tiên, phiên bản BASIC ban đầu của nó chỉ cho phép đặt tên tối đa 6 ký tự. Ký pháp Hungary (KH) đã giúp ích cho việc đặt tên rất nhiều.

KH thực sự được coi là quan trọng khi Windows C API xuất hiện, khi mọi thứ là một số nguyên, một con trỏ kiểu `void` hoặc là các chuỗi,... Trong những ngày đó, trình biên dịch không thể kiểm tra được các lỗi về kiểu dữ liệu, vì vậy các lập trình viên cần một cái phao cứu sinh trong việc nhớ các kiểu dữ liệu này.

Trong các ngôn ngữ hiện đại, chúng ta có nhiều kiểu dữ liệu mặc định hơn, và các trình biên dịch có thể phân biệt được chúng. Hơn nữa, mọi người có xu hướng làm cho các lớp, các hàm trở nên nhỏ hơn để dễ dàng thấy nguồn gốc dữ liệu của biến mà họ đang sử dụng.

Các lập trình viên Java thì không cần mã hóa. Các kiểu dữ liệu mặc định là đủ mạnh mẽ, và các công cụ sửa lỗi đã được nâng cấp để chúng có thể phát hiện các vấn đề về dữ liệu trước khi được biên dịch. Vậy nên, hiện nay KH và các dạng mã hóa khác chỉ đơn giản là một loại chướng ngại vật. Chúng làm cho việc đổi tên biến, tên hàm, tên lớp (hoặc kiểu dữ liệu của chúng) trở nên khó khăn hơn. Chúng làm cho code khó đọc, và tạo ra một hệ thống mã hóa có khả năng đánh lừa người đọc:

```java
PhoneNumber phoneString;
    // name not changed when type changed!
```

### Các thành phần tiền tố

Bạn cũng không cần phải thêm các tiền tố như m\_ vào biến thành viên (member variable) nữa. Các lớp và các hàm phải đủ nhỏ để bạn không cần chúng. Và bạn nên sử dùng các công cụ chỉnh sửa giúp làm nổi bật các biến này, làm cho chúng trở nên khác biệt với phần còn lại.

```java
publicclass Part {
    privateString m_dsc; // The textual description
    void setName(String name) {
    m_dsc = name;
    }
}

/*...*/

publicclass Part {
    String description;
    void setDescription(String description) {
        this.description = description;
    }
}
```

Bên cạnh đó, mọi người cũng nhanh chóng bỏ qua các tiền tố (hoặc hậu tố) để xem phần có ý nghĩa của tên. Càng đọc code, chúng ta càng ít thấy các tiền tố. Cuối cùng, các tiền tố trở nên vô hình, và bị xem là một dấu hiệu của những dòng code lạc hậu.

### Giao diện và thực tiễn

Có một số trường hợp đặc biệt cần mã hóa. Ví dụ: bạn đang xây dựng một ABSTRACT FACTORY. Factory sẽ là giao diện và sẽ được thực hiện bởi một lớp cụ thể. Bạn sẽ đặt tên cho chúng là gì? `IShapeFactory` và `ShapeFactory` ? Tôi thích dùng những cách đặt tên đơn giản. Trước đây, `I` rất phổ biến trong các tài liệu, nó làm chúng tôi phân tâm và đưa ra quá nhiều thông tin. Tôi không muốn người dùng biết rằng tôi đang tạo cho họ một giao diện, tôi chỉ muốn họ biết rằng đó là `ShapeFactory`. Vì vậy, nếu phải lựa chọn việc mã hóa hay thể hiện đầy đủ, tôi sẽ chọn cách thứ nhất. Gọi nó là `ShapeFactoryImp`, hoặc thậm chí là `CShapeFactory` là cách hoàn hảo để che giấu thông tin.

## Tránh "hiếp râm não" người khác

Những lập trình viên khác sẽ không cần phải điên đầu ngồi dịch các tên mà bạn đặt thành những tên mà họ biết. Vấn đề này thường phát sinh khi bạn chọn một thuật ngữ không chính xác.

Đây là vấn đề với các tên biến đơn. Chắc chắn một vong lặp có thể sử dụng các biến được đặt tên là `i`, `j` hoặc `k` (không bao giờ là `l` – dĩ nhiên rồi) nếu phạm vi của nó là rất nhỏ và không có tên khác xung đột với nó. Điều này là do việc đặt tên có một chữ cái trong vòng lặp đã trở thành truyền thống. Tuy nhiên, trong hầu hết trường hợp, tên một chữ cái không phải là sự lựa chọn tốt. Nó chỉ là một tên đầu gấu, bắt người đọc phải điên đầu tìm hiểu ý nghĩa, vai trò của nó. Không có lý do nào tồi tệ hơn cho cho việc sử dụng tên c chỉ vì a và b đã được dùng trước đó.

Nói chung, lập trình viên là những người khá thông minh. Và những người thông minh đôi khi muốn thể hiện điều đó bằng cách hack não người khác. Sau tất cả, nếu bạn đủ khả năng nhớ r là _the lower-cased version of the url with the host and scheme removed,_ thì rõ ràng là – bạn cực kỳ thông minh luôn.

Sự khác biệt giữa lập trình viên thông minh và lập trình viên chuyên nghiệp là họ – những người chuyên nghiệp hiểu rằng sự rõ ràng là trên hết. Các chuyên gia dùng khả năng của họ để tạo nên những dòng code mà người khác có thể hiểu được.

## Tên lớp

Tên lớp và các đối tượng nên sử dụng danh từ hoặc cụm danh từ, như `Customer`, `WikiPage`, `Account`, và `AddressParser`. Tránh những từ như `Manager`, `Processor`, `Data`, hoặc `Info` trong tên của một lớp. Tên lớp không nên dùng động từ.

## Tên các phương thức

Tên các phương thức nên có động từ hoặc cụm động từ như `postPayment`, `deletePage`, hoặc `save`. Các phương thức truy cập, chỉnh sửa thuộc tính phải được đặt tên cùng với `get`, `set` và `is` theo tiêu chuẩn của Javabean.

```java
string name = employee.getName();
customer.setName("mike");
if (paycheck.isPosted())...
```

Khi các hàm khởi tạo bị nạp chồng, sử dụng các phương thức tĩnh có tên thể hiện được đối số sẽ tốt hơn. Ví dụ:

```java
Complex fulcrumPoint = Complex.FromRealNumber(23.0);
```

sẽ tốt hơn câu lệnh

```java
Complex fulcrumPoint = new Complex(23.0);
```

Xem xét việc thực thi chúng bằng các hàm khởi tạo private tương ứng.

## Đừng thể hiện rằng bạn cute

Nếu tên quá hóm hỉnh, chúng sẽ chỉ được nhớ bởi tác giả và những người bạn. Liệu có ai biết chức năng của hàm `HolyHandGrenade` không? Nó rất thú vị, nhưng trong trường hợp này, `DeleteItems` sẽ là tên tốt hơn. Chọn sự rõ ràng thay vì giải trí.

Sự cute thường xuất hiện dưới dạng phong tục hoặc tiếng lóng. Ví dụ: đừng dùng `whack()` thay thế cho `kill()`, đừng mang những câu đùa trong văn hóa nước mình vào code, như `eatMyShorts()` có nghĩa là `abort()`.

_Say what you mean. Mean what you say._

## Chọn một từ cho mỗi khái niệm

Chọn một từ cho một khái niệm và gắn bó với nó. Ví dụ, rất khó hiểu khi `fetch`, `retrieve` và `get` là các phương thức có cùng chức năng, nhưng lại đặt tên khác nhau ở các lớp khác nhau. Làm thế nào để nhớ phương thức nào đi với lớp nào? Buồn thay, bạn phải nhớ tên công ty, nhóm hoặc cá nhân nào đã viết ra các thư viện hoặc các lớp, để nhớ cụm từ nào được dùng cho các phương thức. Nếu không, bạn sẽ mất thời gian để tìm hiểu chúng trong các đoạn code trước đó.

Các công cụ chỉnh sửa hiện đại như Eclipse và IntelliJ cung cấp các định nghĩa theo ngữ cảnh, chẳng hạn như danh sách các hàm bạn có thể gọi trên một đối tượng nhất định. Nhưng lưu ý rằng, danh sách thường không cung cấp cho bạn các ghi chú bạn đã viết xung quanh tên hàm và danh sách tham số. Bạn may mắn nếu nó cung cấp tên tham số từ các khai báo hàm. Tên hàm phải đứng một mình, và chúng phải nhất quán để bạn có thể chọn đúng phương pháp mà không cần phải tìm hiểu thêm.

Tương tự như vậy, rất khó hiểu khi `controller`, `manager` và `driver` lại xuất hiện trong cùng một mã nguồn. Có sự khác biệt nào giữa `DeviceManager` và `ProtocolController`? Tại sao cả hai đều không phải là `controller` hay `manager`? Hay cả hai đều cùng là `driver`? Tên dẫn bạn đến hai đối tượng có kiểu khác nhau, cũng như có các lớp khác nhau.

Một từ phù hợp chính là một ân huệ cho những lập trình viên phải dùng code của bạn.

## Đừng chơi chữ

Tránh dùng cùng một từ cho hai mục đích. Sử dụng cùng một thuật ngữ cho hai ý tưởng khác nhau về cơ bản là một cách chơi chữ.

Nếu bạn tuân theo nguyên tắc _Chọn một từ cho mỗi khái niệm,_ bạn có thể kết thúc nhiều lớp với một... Ví dụ, phương thức add. Miễn là danh sách tham số và giá trị trả về của các phương thức add này tương đương về ý nghĩa, tất cả đều tốt.

Tuy nhiên, người ta có thể quyết định dùng từ add khi người đó không thực sự tạo nên một hàm có cùng ý nghĩa với cách hoạt động của hàm `add`. Giả sử chúng tôi có nhiều lớp, trong đó `add` sẽ tạo một giá trị mới bằng cách cộng hoặc ghép hai giá trị hiện tại. Bây giờ, giả sử chúng tôi đang viết một lớp mới và có một phương thức thêm tham số của nó vào mảng. Chúng tôi có nên gọi nó là `add` không? Có vẻ phù hợp đấy, nhưng trong trường hợp này, ý nghĩa của chúng là khác nhau, vậy nên chúng tôi dùng một cái tên khác như `insert` hay `append` để thay thế. Nếu được dùng cho phương thức mới, add chính xác là một kiểu chơi chữ.

Mục tiêu của chúng tôi, với tư cách là tác giả, là làm cho code của chúng tôi dễ hiểu nhất có thể. Chúng tôi muốn code của chúng tôi là một bài viết ngắn gọn, chứ không phải là một bài nghiên cứu [...].

## Dùng thuật ngữ

Hãy nhớ rằng những người đọc code của bạn là những lập trình viên, vậy nên hãy sử dụng các thuật ngữ khoa học, các thuật toán, tên mẫu (pattern),... cho việc đặt tên. Sẽ không khôn ngoan khi bạn đặt tên của vấn đề theo cách mà khách hàng định nghĩa. Chúng tôi không muốn đồng nghiệp của chúng tôi phải tìm khách hàng để hỏi ý nghĩa của tên, trong khi họ đã biết khái niệm đó – nhưng là dưới dạng một cái tên khác.

Tên `AccountVisitor` có ý nghĩa rất nhiều đối với một lập trình viên quen thuộc với mô hình VISITOR (VISITOR pattern). Có lập trình viên nào không biết `JobQueue`? Có rất nhiều thứ liên quan đến kỹ thuật mà lập trình viên phải đặt tên. Chọn những tên thuật ngữ thường là cách tốt nhất.

## Thêm ngữ cảnh thích hợp

Chỉ có một vài cái tên có nghĩa trong mọi trường hợp – số còn lại thì không. Vậy nên, bạn cần đặt tên phù hợp với ngữ cảnh, bằng cách đặt chúng vào các lớp, các hàm hoặc các không gian tên (namespace). Khi mọi thứ thất bại, tiền tố nên được cân nhắc như là giải pháp cuối cùng.

Hãy tưởng tượng bạn có các biến có tên là `firstName`, `lastName`, `street`, `houseNumber`, `city`, `state` và `zipcode`. Khi kết hợp với nhau, chúng rõ ràng tạo thành một địa chỉ. Nhưng nếu bạn chỉ thấy biến state được sử dụng một mình trong một phương thức thì sao? Bạn có thể suy luận ra đó là một phần của địa chỉ không?

Bạn có thể thêm ngữ cảnh bằng cách sử dụng tiền tố: `addrFirstName`, `addrLastName`, `addrState`,... Ít nhất người đọc sẽ hiểu rằng những biến này là một phần của một cấu trúc lớn hơn. Tất nhiên, một giải pháp tốt hơn là tạo một lớp có tên là `Address`. Khi đó, ngay cả trình biên dịch cũng biết rằng các biến đó thuộc về một khái niệm lớn hơn.

Hãy xem xét các phương thức trong _Listing 2-1_. Các biến có cần một ngữ cảnh có ý nghĩa hơn không? Tên hàm chỉ cung cấp một phần của ngữ cảnh, thuật toán cung cấp phần còn lại. Khi bạn đọc qua hàm, bạn thấy rằng ba biến, `number`, `verb` và `pluralModifier`, là một phần của thông báo "giả định thống kê". Thật không may, bối cảnh này phải suy ra mới có được. Khi bạn lần đầu xem xét phương thức, ý nghĩa của các biến là không rõ ràng.

**\# Listing 2-1: Biến với bối cảnh không rõ ràng**

```java
private void printGuessStatistics(char candidate, int count) {
    String number;
    String verb;
    String pluralModifier;
    if (count == 0) {
        number = "no";
        verb = "are";
        pluralModifier = "s";
    } else if (count == 1) {
        number = "1";
        verb = "is";
        pluralModifier = "";
    } else {
        number = Integer.toString(count);
        verb = "are";
        pluralModifier = "s";
    }
    String guessMessage = String.format("There %s %s %s%s", verb, number, candidate, pluralModifier);
    print(guessMessage);
}
```

Hàm này hơi dài và các biến được sử dụng xuyên suốt. Để tách hàm thành các phần nhỏ hơn, chúng ta cần tạo một lớp `GuessStatisticsMessage` và tạo ra ba biến của lớp này. Điều này cung cấp một bối cảnh rõ ràng cho ba biến. Chúng là một phần của `GuessStatisticsMessage`. Việc cải thiện bối cảnh cũng cho phép thuật toán được rõ ràng hơn bằng cách chia nhỏ nó thành nhiều chức năng nhỏ hơn. (Xem _Listing 2-2_.)

**\# Listing 2-2: Biến có ngữ cảnh**

```java
public class GuessStatisticsMessage {
    private String number;
    private String verb;
    private String pluralModifier;
    public String make(char candidate, int count) {
        createPluralDependentMessageParts(count);
        return String.format("There %s %s %s%s",verb, number, candidate, pluralModifier );
    }
    private void createPluralDependentMessageParts(int count) {
        if (count == 0) {
            thereAreNoLetters();
        } else if (count == 1) {
            thereIsOneLetter();
        } else {
            thereAreManyLetters(count);
        }
    }	
    private void thereAreManyLetters(int count) {
        number = Integer.toString(count);
        verb = "are";
        pluralModifier = "s";
    }
    private void thereIsOneLetter() {
        number = "1";
        verb = "is";
        pluralModifier = "";
    }
    private void thereAreNoLetters() {
        number = "no";
        verb = "are";
        pluralModifier = "s";
    }
}

```

Tên ngắn thường tốt hơn tên dài, miễn là chúng rõ ràng. Thêm đủ ngữ cảnh cho tên sẽ tốt hơn khi cần thiết.

Tên `accountAddress` và `customerAddress` là những tên đẹp cho trường hợp đặc biệt của lớp `Address` nhưng có thể là tên tồi cho các lớp khác. `Address` là một tên đẹp cho lớp. Nếu tôi cần phân biệt giữa địa chỉ MAC, địa chỉ cổng (port) và địa chỉ web thì tôi có thể xem xét `MAC`, `PostalAddress` và `URL`. Kết quả là tên chính xác hơn. Đó là tâm điểm của việc đặt tên.

## Lời kết

Điều khó khăn nhất của việc lựa chọn tên đẹp là nó đòi hỏi kỹ năng mô tả tốt và nền tảng văn hóa lớn. Đây là vấn đề về học hỏi hơn là vấn đề kỹ thuật, kinh doanh hoặc quản lý. Kết quả là nhiều người trong lĩnh vực này không học cách làm điều đó.

Mọi người cũng sợ đổi tên mọi thứ vì lo rằng người khác sẽ phản đối. Chúng tôi không chia sẻ nỗi sợ đó cho bạn. Chúng tôi thật sự biết ơn những ai đã đổi tên khác cho biến, hàm,...(theo hướng tốt hơn). Hầu hết thời gian chúng tôi không thật sự nhớ tên lớp và những phương thức của nó. Chúng tôi có các công cụ giúp chúng tôi trong việc đó để chúng tôi có thể tập trung vào việc code có dễ đọc hay không. Bạn có thể sẽ gây ngạc nhiên cho ai đó khi bạn đổi tên, giống như bạn có thể làm với bất kỳ cải tiến nào khác. Đừng để những cái tên tồi phá hủy sự nghiệp coder của mình.

Thực hiện theo một số quy tắc trên và xem liệu bạn có cải thiện được khả năng đọc code của mình hay không. Nếu bạn đang bảo trì code của người khác, hãy sử dụng các công cụ tái cấu trúc để giải quyết vấn đề này. Mất một ít thời gian nhưng có thể làm bạn nhẹ nhõm trong vài tháng.

_- Viết bởi Tim Ottinger_

## Giới thiệu

Những cái tên có ở khắp mọi nơi trong phần mềm. Chúng ta đặt tên cho các biến, các hàm, các đối số, các lớp và các gói của chúng ta. Chúng ta đặt tên cho những file mã nguồn và thư mục chứa chúng. Chúng ta đặt tên cho những file _\*.jar_, file _\*.war,.._. Chúng ta đặt tên và đặt tên. Vì chúng ta đặt tên rất nhiều, nên chúng ta cần làm tốt điều đó. Sau đây là một số quy tắc đơn giản để tạo nên những cái tên tốt.

## Dùng những tên thể hiện được mục đích

Điều này rất dễ. Nhưng chúng tôi muốn nhấn mạnh rằng chúng tôi nghiêm túc trong việc này. Chọn một cái tên "xịn" mất khá nhiều thời gian, nhưng lại tiết kiệm (thời gian) hơn sau đó. Vì vậy, hãy quan tâm đến cái tên mà bạn chọn và chỉ thay đổi chúng khi bạn sáng tạo ra tên "xịn" hơn. Những người đọc code của bạn (kể cả bạn) sẽ _sung sướng_ hơn khi bạn làm điều đó.

Tên của biến, hàm, hoặc lớp phải trả lời tất cả những câu hỏi về nó. Nó phải cho bạn biết lý do nó tồn tại, nó làm được những gì, và dùng nó ra sao. Nếu có một comment đi kèm theo tên, thì tên đó không thể hiện được mục đích của nó.

```java
int d; // elapsed time in days
```

Tên **d** không tiết lộ điều gì cả. Nó không gợi lên cảm giác gì về thời gian, cũng không liên quan gì đến ngày. Chúng ta nên chọn một tên thể hiện được những gì đang được cân đo, và cả đơn vị đo của chúng:

```java
int elapsedTimeInDays;
int daysSinceCreation;
int daysSinceModification;
int fileAgeInDays;
```

Việc chọn tên thể hiện được mục đích có thể làm cho việc hiểu và thay đổi code dễ dàng hơn nhiều. Hãy đoán xem mục đích của đoạn code dưới đây là gì?

```java
public List<int[]> getThem() {
    List<int[]> list1 = newArrayList<int[]>();
    for (int[] x : theList)
        if (x[0] == 4)
            list1.add(x);
    return list1;
}
```

Tại sao lại nói khó mà biết được đoạn code này đang làm gì? Không có biểu thức phức tạp, khoảng cách và cách thụt đầu dòng hợp lý, chỉ có 3 biến và 2 hằng số được đề cập. Thậm chí không có các lớp (class) và phương thức đa hình nào, nó chỉ có một danh sách mảng (hoặc thứ gì đó trông giống vậy).

Vấn đề không nằm ở sự đơn giản của code mà nằm ở ý nghĩa của code, do bối cảnh không rõ ràng. Đoạn code trên bắt chúng tôi phải tìm câu trả lời cho các câu hỏi sau:

1. theList chứa cái gì?
2. Ý nghĩa của chỉ số 0 trong phần tử của theList?
3. Số 4 có ý nghĩa gì?
4. Danh sách được return thì dùng kiểu gì?

Câu trả lời không có trong code, nhưng sẽ có ngay sau đây. Giả sử chúng tôi đang làm game _dò mìn_. Chúng tôi thấy rằng giao diện trò chơi là một danh sách các ô vuông (cell) được gọi là theList. Vậy nên, hãy đổi tên nó thành gameBoard.

Mỗi ô trên màn hình được biểu diễn bằng một sanh sách đơn giản. Chúng tôi cũng thấy rằng chỉ số của số 0 là vị trí biểu diễn giá trị trạng thái (status value), và giá trị 4 nghĩa là trạng thái _được gắn cờ (flagged)._ Chỉ bằng cách đưa ra các khái niệm này, chúng tôi có thể cải thiện mã nguồn một cách đáng kể:

```java
public List<int[]> getFlaggedCells() {
    List<int[]> flaggedCells = newArrayList<int[]>();
    for (int[] cell : gameBoard)
        if (cell[STATUS_VALUE] == FLAGGED)
            flaggedCells.add(cell);
    return flaggedCells;
}
```

Cần lưu ý rằng mức độ đơn giản của code vẫn không thay đổi, nó vẫn chính xác về toán tử, hằng số, và các lệnh lồng nhau,…Nhưng đã trở nên rõ ràng hơn rất nhiều.

Chúng ta có thể đi xa hơn bằng cách viết một lớp đơn giản cho các ô thay vì sử dụng các mảng kiểu int. Nó có thể bao gồm một hàm thể hiện được mục đích (gọi nó là _isFlagged – được gắn cờ_ chẳng hạn) để giấu đi những con số ma thuật _(Từ gốc: magic number – Một khái niệm về các hằng số, tìm hiểu thêm tại_ [https://en.wikipedia.org/wiki/Magic_number_(programming)](https://en.wikipedia.org/wiki/Magic_number_(programming)) _)._

```java
public List<Cell> getFlaggedCells() {
    List<Cell> flaggedCells = newArrayList<Cell>();
    for (Cell cell : gameBoard)
        if (cell.isFlagged())
            flaggedCells.add(cell);
    return flaggedCells;
}
```

Với những thay đổi đơn giản này, không quá khó để hiểu những gì mà đoạn code đang trình bày. Đây chính là sức mạnh của việc chọn tên tốt.

## Tránh sai lệch thông tin

Các lập trình viên phải tránh để lại những dấu hiệu làm code trở nên khó hiểu. Chúng ta nên tránh dùng những từ mang nghĩa khác với nghĩa cố định của nó. Ví dụ, các tên biến như `hp`, `aix` và `sco` là những tên biến vô cùng tồi tệ, chúng là tên của các nền tảng Unix hoặc các biến thể. Ngay cả khi bạn đang code về cạnh huyền (hypotenuse) và hptrông giống như một tên viết tắt tốt, rất có thể đó là một cái tên tồi.

Không nên quy kết rằng một nhóm các tài khoản là một `accountList` nếu nó không thật sự là một danh sách (`List`). Từ _danh sách_ có nghĩa là một thứ gì đó cụ thể cho các lập trình viên. Nếu các tài khoản không thực sự tạo thành danh sách, nó có thể dẫn đến một kết quả sai lầm. Vậy nên, `accountGroup` hoặc `bunchOfAccounts`, hoặc đơn giản chỉ là accounts sẽ tốt hơn.

Cẩn thận với những cái tên gần giống nhau. Mất bao lâu để bạn phân biệt được sự khác nhau giữa `XYZControllerForEfficientHandlingOfStrings` và `XYZControllerForEfficientStorageOfStrings` trong cùng một module, hay đâu đó xa hơn một chút? Những cái tên gần giống nhau như thế này thật sự, thật sự rất khủng khiếp cho lập trình viên.

Một kiểu khủng bố tinh thần khác về những cái tên không rõ ràng là ký tự L viết thường và O viết hoa. Vấn đề? Tất nhiên là nhìn chúng gần như hoàn toàn giống hằng số không và một, kiểu như:

```
int a = l;  
if ( O == l ) a = O1;  
else l = 01;
```

Bạn nghĩ chúng tôi _xạo_? Chúng tôi đã từng khảo sát, và kiểu code như vậy thực sự rất nhiều. Trong một số trường hợp, tác giả của code đề xuất sử dụng phông chữ khác nhau để tách biệt chúng. Một giải pháp khác có thể được sử dụng là truyền đạt bằng lời nói hoặc để lại tài liệu cho các lập trình viên sau này có thể hiểu nó. Vấn đề được giải quyết mà không cần phải đổi tên để tạo ra một sản phẩm khác.

## Tạo nên những sự khác biệt có nghĩa

Các lập trình viên tạo ra vấn đề cho chính họ khi viết code chỉ để đáp ứng cho trình biên dịch hoặc thông dịch. Ví dụ, vì bạn không thể sử dụng cùng một tên để chỉ hai thứ khác nhau trong cùng một khối lệnh hoặc cùng một phạm vi, bạn có thể bị "dụ dỗ" thay đổi tên một cách tùy tiện. Đôi khi điều đó làm bạn cố tình viết sai chính tả, và người nào đó quyết định sửa lỗi chính tả đó, khiến trình biên dịch không có khả năng hiểu nó (cụ thể – tạo ra một biến tên _klass_ chỉ vì tên _class_ đã được dùng cho thứ gì đó).

Mặc dù trình biên dịch có thể làm việc với những tên này, nhưng điều đó không có nghĩa là bạn được phép dùng nó. Nếu tên khác nhau, thì chúng cũng có ý nghĩa khác nhau.

Những tên dạng chuỗi số (a1, a2,… aN) đi ngược lại nguyên tắc đặt tên có mục đích. Mặc dù những tên như vậy không phải là không đúng, nhưng chúng không có thông tin. Chúng không cung cấp manh mối nào về ý định của tác giả. Ví dụ:

```java
public static void copyChars(char a1[], char a2[]) {
    for (int i = 0; i < a1.length; i++) {
        a2[i] = a1[i];
    }
}
```

Hàm này dễ đọc hơn nhiều khi _nguyên nhân_ và _mục đích_ của nó được đặt tên cho các đối số.

Những từ gây nhiễu tạo nên sự khác biệt, nhưng là sự khác biệt vô dụng. Hãy tưởng tượng rằng bạn có một lớp `Product`, nếu bạn có một `ProductInfo` hoặc `ProductData` khác, thì bạn đã thành công trong việc tạo ra các tên khác nhau nhưng về mặt ngữ nghĩa thì chúng là một. `Info` và `Data` là các từ gây nhiễu, giống như `a`, `an` và `the`.

Lưu ý rằng không có gì sai khi sử dụng các tiền tố như `a` và `the` để tạo ra những khác biệt hữu ích. Ví dụ, bạn có thể sử dụng `a` cho tất cả các biến cục bộ và tất cả các đối số của hàm. `a` và `the` sẽ trở thành vô dụng khi bạn quyết định tạo một biến `theZork` vì trước đó bạn đã có một biến mang tên `Zork`.

Những từ gây nhiễu là không cần thiết. Từ `variable` sẽ không bao giờ xuất hiện trong tên biến, từ `table` cũng không nên dùng trong tên bảng. `NameString` sao lại tốt hơn `Name`? `Name` có bao giờ là một số đâu mà lại? Nếu `Name` là một số, nó đã phá vỡ nguyên tắc _Tránh sai lệch thông tin._ Hãy tưởng tượng bạn đang tìm kiếm một lớp có tên `Customer`, và một lớp khác có tên `CustomerObject`. Chúng khác nhau kiểu gì? Cái nào chứa lịch sử thanh toán của khách hàng? Còn cái nào chứa thông tin của khách?

Có một ứng dụng minh họa cho các lỗi trên, chúng tôi đã thay đổi một chút về tên để bảo vệ tác giả. Đây là những thứ chúng tôi thấy trong mã nguồn:

```java
getActiveAccount();
getActiveAccounts();
getActiveAccountInfo();
```

Tôi thắc mắc không biết các lập trình viên trong dự án này phải `getActiveAccount` như thế nào!

Trong trường hợp không có quy ước cụ thể, biến `moneyAmount` không thể phân biệt được với `money`; `customerInfo` không thể phân biệt được với `customer`; `accountData` không thể phân biệt được với `account` và `theMessage` với `message` được xem là một. Hãy phân biệt tên theo cách cung cấp cho người đọc những khác biệt rõ ràng.

## Dùng những tên phát âm được

Con người rất giỏi về từ ngữ. Một phần quan trọng trong bộ não của chúng ta được dành riêng cho các khái niệm về từ. Và các từ, theo định nghĩa, có thể phát âm được. Thật lãng phí khi không sử dụng được bộ não mà chúng ta đã tiến hóa nhằm thích nghi với ngôn ngữ nói. Vậy nên, hãy làm cho những cái tên phát âm được đi nào.

Nếu bạn không thể phát âm nó, thì bạn không thể thảo luận một cách bình thường: "Hey, ở đây chúng ta có _bee cee arr three cee enn tee_, và _pee ess zee kyew int_, thấy chứ?" – Vâng, tôi thấy một thằng thiểu năng. Vấn đề này rất quan trọng vì lập trình cũng là một hoạt động xã hội, chúng ta cần trao đổi với mọi người.

Tôi có biết một công ty dùng tên _genymdhms_ (generation date, year, month, day, hour, minute, and second – phát sinh ngày, tháng, năm, giờ, phút, giây), họ đi xung quanh tôi và "gen why emm dee aich emm ess" (cách phát âm theo tiếng Anh). Tôi có thói quen phát âm như những gì tôi viết, vì vậy tôi bắt đầu nói "gen-yah-muddahims". Sau này nó được gọi bởi một loạt các nhà thiết kế và phân tích, và nghe vẫn có vẻ ngớ ngẫn. Chúng tôi đã từng troll nhau như thế, nó rất thú vị. Nhưng dẫu thế nào đi nữa, chúng tôi đã chấp nhận những cái tên xấu xí. Những lập trình viên mới của công ty tìm hiểu ý nghĩa của các biến, và sau đó họ nói về những từ ngớ ngẫn, thay vì dùng các thuật ngữ tiếng Anh cho thích hợp. Hãy so sánh:

```java
class DtaRcrd102 {
    privateDate genymdhms;
    privateDate modymdhms;
    privatefinalString pszqint = "102";
    /* ... */
};
```

và

```java
class Customer {
    privateDate generationTimestamp;
    privateDate modificationTimestamp;
    privatefinalString recordId = "102";
    /* ... */
};
```

Cuộc trò chuyện giờ đây đã thông minh hơn: "Hey, Mikey, take a look at this record! The generation timestamp is set to tomorrow's date! How can that be?"

## Dùng những tên tìm kiếm được

Các tên một chữ cái và các hằng số luôn có vấn đề, đó là không dễ để tìm chúng trong hàng ngàn dòng code.

Người ta có thể dễ dàng tìm kiếm `MAX_CLASSES_PER_STUDENT`, nhưng số 7 thì lại rắc rối hơn. Các công cụ tìm kiếm có thể mở các tệp, các hằng, hoặc các biểu thức chứa số 7 này, nhưng được sử dụng với các mục đích khác nhau. Thậm chí còn tồi tệ hơn khi hằng số là một số có giá trị lớn và ai đó vô tình thay đổi giá trị của nó, từ đó tạo ra một lỗi mà các lập trình viên không tìm ra được.

Tương tự như vậy, tên e là một sự lựa chọn tồi tệ cho bất kỳ biến nào mà một lập trình viên cần tìm kiếm. Nó là chữ cái phổ biến nhất trong tiếng anh và có khả năng xuất hiện trong mọi đoạn code của chương trình. Về vấn đề này, tên dài thì tốt hơn tên ngắn, và những cái tên tìm kiếm được sẽ tốt hơn một hằng số trơ trọi trong code.

Sở thích cá nhân của tôi là chỉ đặt tên ngắn cho những biến cục bộ bên trong những phương thức ngắn. _Độ dài của tên phải tương ứng với phạm vi hoạt động của nó_. Nếu một biến hoặc hằng số được nhìn thấy và sử dụng ở nhiều vị trí trong phần thân của mã nguồn, bắt buộc phải đặt cho nó một tên dễ tìm kiếm. Ví dụ:

```java
for (int j=0; j<34; j++) {
    s += (t[j]*4)/5;
}
```

và

```java
int realDaysPerIdealDay = 4;
constint WORK_DAYS_PER_WEEK = 5;
int sum = 0;
for (int j=0; j < NUMBER_OF_TASKS; j++) {
    int realTaskDays = taskEstimate[j] * realDaysPerIdealDay;
    int realTaskWeeks = (realdays / WORK_DAYS_PER_WEEK);
    sum += realTaskWeeks;
}
```

Lưu ý rằng sum ở ví dụ trên, dù không phải là một tên đầy đủ nhưng có thể tìm kiếm được. Biến và hằng được cố tình đặt tên dài, nhưng hãy so sánh việc tìm kiếm WORK\_DAYS\_PER\_WEEK dễ hơn bao nhiêu lần so với số 5, đó là chưa kể cần phải lọc lại danh sách tìm kiếm và tìm ra những trường hợp có nghĩa.

## Tránh việc mã hóa

Các cách mã hóa hiện tại là đủ với chúng tôi. Mã hóa các kiểu dữ liệu hoặc phạm vi thông tin vào tên chỉ đơn giản là thêm một gánh nặng cho việc giải mã. Điều đó không hợp lý khi bắt nhân viên phải học thêm một "ngôn ngữ" mã hóa khác khác ngoài các ngôn ngữ mà họ dùng để làm việc với code. Đó là một gánh nặng không cần thiết, các tên mã hóa ít khi được phát âm và dễ bị đánh máy sai.

### Ký pháp Hungary

Ngày trước, khi chúng tôi làm việc với những ngôn ngữ mà độ dài tên là một thách thức, chúng tôi đã loại bỏ sự cần thiết này. Fortran bắt buộc mã hóa bằng những chữ cái đầu tiên, phiên bản BASIC ban đầu của nó chỉ cho phép đặt tên tối đa 6 ký tự. Ký pháp Hungary (KH) đã giúp ích cho việc đặt tên rất nhiều.

KH thực sự được coi là quan trọng khi Windows C API xuất hiện, khi mọi thứ là một số nguyên, một con trỏ kiểu `void` hoặc là các chuỗi,... Trong những ngày đó, trình biên dịch không thể kiểm tra được các lỗi về kiểu dữ liệu, vì vậy các lập trình viên cần một cái phao cứu sinh trong việc nhớ các kiểu dữ liệu này.

Trong các ngôn ngữ hiện đại, chúng ta có nhiều kiểu dữ liệu mặc định hơn, và các trình biên dịch có thể phân biệt được chúng. Hơn nữa, mọi người có xu hướng làm cho các lớp, các hàm trở nên nhỏ hơn để dễ dàng thấy nguồn gốc dữ liệu của biến mà họ đang sử dụng.

Các lập trình viên Java thì không cần mã hóa. Các kiểu dữ liệu mặc định là đủ mạnh mẽ, và các công cụ sửa lỗi đã được nâng cấp để chúng có thể phát hiện các vấn đề về dữ liệu trước khi được biên dịch. Vậy nên, hiện nay KH và các dạng mã hóa khác chỉ đơn giản là một loại chướng ngại vật. Chúng làm cho việc đổi tên biến, tên hàm, tên lớp (hoặc kiểu dữ liệu của chúng) trở nên khó khăn hơn. Chúng làm cho code khó đọc, và tạo ra một hệ thống mã hóa có khả năng đánh lừa người đọc:

```java
PhoneNumber phoneString;
    // name not changed when type changed!
```

### Các thành phần tiền tố

Bạn cũng không cần phải thêm các tiền tố như m\_ vào biến thành viên (member variable) nữa. Các lớp và các hàm phải đủ nhỏ để bạn không cần chúng. Và bạn nên sử dùng các công cụ chỉnh sửa giúp làm nổi bật các biến này, làm cho chúng trở nên khác biệt với phần còn lại.

```java
publicclass Part {
    privateString m_dsc; // The textual description
    void setName(String name) {
    m_dsc = name;
    }
}

/*...*/

publicclass Part {
    String description;
    void setDescription(String description) {
        this.description = description;
    }
}
```

Bên cạnh đó, mọi người cũng nhanh chóng bỏ qua các tiền tố (hoặc hậu tố) để xem phần có ý nghĩa của tên. Càng đọc code, chúng ta càng ít thấy các tiền tố. Cuối cùng, các tiền tố trở nên vô hình, và bị xem là một dấu hiệu của những dòng code lạc hậu.

### Giao diện và thực tiễn

Có một số trường hợp đặc biệt cần mã hóa. Ví dụ: bạn đang xây dựng một ABSTRACT FACTORY. Factory sẽ là giao diện và sẽ được thực hiện bởi một lớp cụ thể. Bạn sẽ đặt tên cho chúng là gì? `IShapeFactory` và `ShapeFactory` ? Tôi thích dùng những cách đặt tên đơn giản. Trước đây, `I` rất phổ biến trong các tài liệu, nó làm chúng tôi phân tâm và đưa ra quá nhiều thông tin. Tôi không muốn người dùng biết rằng tôi đang tạo cho họ một giao diện, tôi chỉ muốn họ biết rằng đó là `ShapeFactory`. Vì vậy, nếu phải lựa chọn việc mã hóa hay thể hiện đầy đủ, tôi sẽ chọn cách thứ nhất. Gọi nó là `ShapeFactoryImp`, hoặc thậm chí là `CShapeFactory` là cách hoàn hảo để che giấu thông tin.

## Tránh "hiếp râm não" người khác

Những lập trình viên khác sẽ không cần phải điên đầu ngồi dịch các tên mà bạn đặt thành những tên mà họ biết. Vấn đề này thường phát sinh khi bạn chọn một thuật ngữ không chính xác.

Đây là vấn đề với các tên biến đơn. Chắc chắn một vong lặp có thể sử dụng các biến được đặt tên là `i`, `j` hoặc `k` (không bao giờ là `l` – dĩ nhiên rồi) nếu phạm vi của nó là rất nhỏ và không có tên khác xung đột với nó. Điều này là do việc đặt tên có một chữ cái trong vòng lặp đã trở thành truyền thống. Tuy nhiên, trong hầu hết trường hợp, tên một chữ cái không phải là sự lựa chọn tốt. Nó chỉ là một tên đầu gấu, bắt người đọc phải điên đầu tìm hiểu ý nghĩa, vai trò của nó. Không có lý do nào tồi tệ hơn cho cho việc sử dụng tên c chỉ vì a và b đã được dùng trước đó.

Nói chung, lập trình viên là những người khá thông minh. Và những người thông minh đôi khi muốn thể hiện điều đó bằng cách hack não người khác. Sau tất cả, nếu bạn đủ khả năng nhớ r là _the lower-cased version of the url with the host and scheme removed,_ thì rõ ràng là – bạn cực kỳ thông minh luôn.

Sự khác biệt giữa lập trình viên thông minh và lập trình viên chuyên nghiệp là họ – những người chuyên nghiệp hiểu rằng sự rõ ràng là trên hết. Các chuyên gia dùng khả năng của họ để tạo nên những dòng code mà người khác có thể hiểu được.

## Tên lớp

Tên lớp và các đối tượng nên sử dụng danh từ hoặc cụm danh từ, như `Customer`, `WikiPage`, `Account`, và `AddressParser`. Tránh những từ như `Manager`, `Processor`, `Data`, hoặc `Info` trong tên của một lớp. Tên lớp không nên dùng động từ.

## Tên các phương thức

Tên các phương thức nên có động từ hoặc cụm động từ như `postPayment`, `deletePage`, hoặc `save`. Các phương thức truy cập, chỉnh sửa thuộc tính phải được đặt tên cùng với `get`, `set` và `is` theo tiêu chuẩn của Javabean.

```java
string name = employee.getName();
customer.setName("mike");
if (paycheck.isPosted())...
```

Khi các hàm khởi tạo bị nạp chồng, sử dụng các phương thức tĩnh có tên thể hiện được đối số sẽ tốt hơn. Ví dụ:

```java
Complex fulcrumPoint = Complex.FromRealNumber(23.0);
```

sẽ tốt hơn câu lệnh

```java
Complex fulcrumPoint = new Complex(23.0);
```

Xem xét việc thực thi chúng bằng các hàm khởi tạo private tương ứng.

## Đừng thể hiện rằng bạn cute

Nếu tên quá hóm hỉnh, chúng sẽ chỉ được nhớ bởi tác giả và những người bạn. Liệu có ai biết chức năng của hàm `HolyHandGrenade` không? Nó rất thú vị, nhưng trong trường hợp này, `DeleteItems` sẽ là tên tốt hơn. Chọn sự rõ ràng thay vì giải trí.

Sự cute thường xuất hiện dưới dạng phong tục hoặc tiếng lóng. Ví dụ: đừng dùng `whack()` thay thế cho `kill()`, đừng mang những câu đùa trong văn hóa nước mình vào code, như `eatMyShorts()` có nghĩa là `abort()`.

_Say what you mean. Mean what you say._

## Chọn một từ cho mỗi khái niệm

Chọn một từ cho một khái niệm và gắn bó với nó. Ví dụ, rất khó hiểu khi `fetch`, `retrieve` và `get` là các phương thức có cùng chức năng, nhưng lại đặt tên khác nhau ở các lớp khác nhau. Làm thế nào để nhớ phương thức nào đi với lớp nào? Buồn thay, bạn phải nhớ tên công ty, nhóm hoặc cá nhân nào đã viết ra các thư viện hoặc các lớp, để nhớ cụm từ nào được dùng cho các phương thức. Nếu không, bạn sẽ mất thời gian để tìm hiểu chúng trong các đoạn code trước đó.

Các công cụ chỉnh sửa hiện đại như Eclipse và IntelliJ cung cấp các định nghĩa theo ngữ cảnh, chẳng hạn như danh sách các hàm bạn có thể gọi trên một đối tượng nhất định. Nhưng lưu ý rằng, danh sách thường không cung cấp cho bạn các ghi chú bạn đã viết xung quanh tên hàm và danh sách tham số. Bạn may mắn nếu nó cung cấp tên tham số từ các khai báo hàm. Tên hàm phải đứng một mình, và chúng phải nhất quán để bạn có thể chọn đúng phương pháp mà không cần phải tìm hiểu thêm.

Tương tự như vậy, rất khó hiểu khi `controller`, `manager` và `driver` lại xuất hiện trong cùng một mã nguồn. Có sự khác biệt nào giữa `DeviceManager` và `ProtocolController`? Tại sao cả hai đều không phải là `controller` hay `manager`? Hay cả hai đều cùng là `driver`? Tên dẫn bạn đến hai đối tượng có kiểu khác nhau, cũng như có các lớp khác nhau.

Một từ phù hợp chính là một ân huệ cho những lập trình viên phải dùng code của bạn.

## Đừng chơi chữ

Tránh dùng cùng một từ cho hai mục đích. Sử dụng cùng một thuật ngữ cho hai ý tưởng khác nhau về cơ bản là một cách chơi chữ.

Nếu bạn tuân theo nguyên tắc _Chọn một từ cho mỗi khái niệm,_ bạn có thể kết thúc nhiều lớp với một... Ví dụ, phương thức add. Miễn là danh sách tham số và giá trị trả về của các phương thức add này tương đương về ý nghĩa, tất cả đều tốt.

Tuy nhiên, người ta có thể quyết định dùng từ add khi người đó không thực sự tạo nên một hàm có cùng ý nghĩa với cách hoạt động của hàm `add`. Giả sử chúng tôi có nhiều lớp, trong đó `add` sẽ tạo một giá trị mới bằng cách cộng hoặc ghép hai giá trị hiện tại. Bây giờ, giả sử chúng tôi đang viết một lớp mới và có một phương thức thêm tham số của nó vào mảng. Chúng tôi có nên gọi nó là `add` không? Có vẻ phù hợp đấy, nhưng trong trường hợp này, ý nghĩa của chúng là khác nhau, vậy nên chúng tôi dùng một cái tên khác như `insert` hay `append` để thay thế. Nếu được dùng cho phương thức mới, add chính xác là một kiểu chơi chữ.

Mục tiêu của chúng tôi, với tư cách là tác giả, là làm cho code của chúng tôi dễ hiểu nhất có thể. Chúng tôi muốn code của chúng tôi là một bài viết ngắn gọn, chứ không phải là một bài nghiên cứu […].

## Dùng thuật ngữ

Hãy nhớ rằng những người đọc code của bạn là những lập trình viên, vậy nên hãy sử dụng các thuật ngữ khoa học, các thuật toán, tên mẫu (pattern),... cho việc đặt tên. Sẽ không khôn ngoan khi bạn đặt tên của vấn đề theo cách mà khách hàng định nghĩa. Chúng tôi không muốn đồng nghiệp của chúng tôi phải tìm khách hàng để hỏi ý nghĩa của tên, trong khi họ đã biết khái niệm đó – nhưng là dưới dạng một cái tên khác.

Tên `AccountVisitor` có ý nghĩa rất nhiều đối với một lập trình viên quen thuộc với mô hình VISITOR (VISITOR pattern). Có lập trình viên nào không biết `JobQueue`? Có rất nhiều thứ liên quan đến kỹ thuật mà lập trình viên phải đặt tên. Chọn những tên thuật ngữ thường là cách tốt nhất.

## Thêm ngữ cảnh thích hợp

Chỉ có một vài cái tên có nghĩa trong mọi trường hợp – số còn lại thì không. Vậy nên, bạn cần đặt tên phù hợp với ngữ cảnh, bằng cách đặt chúng vào các lớp, các hàm hoặc các không gian tên (namespace). Khi mọi thứ thất bại, tiền tố nên được cân nhắc như là giải pháp cuối cùng.

Hãy tưởng tượng bạn có các biến có tên là `firstName`, `lastName`, `street`, `houseNumber`, `city`, `state` và `zipcode`. Khi kết hợp với nhau, chúng rõ ràng tạo thành một địa chỉ. Nhưng nếu bạn chỉ thấy biến state được sử dụng một mình trong một phương thức thì sao? Bạn có thể suy luận ra đó là một phần của địa chỉ không?

Bạn có thể thêm ngữ cảnh bằng cách sử dụng tiền tố: `addrFirstName`, `addrLastName`, `addrState`,... Ít nhất người đọc sẽ hiểu rằng những biến này là một phần của một cấu trúc lớn hơn. Tất nhiên, một giải pháp tốt hơn là tạo một lớp có tên là `Address`. Khi đó, ngay cả trình biên dịch cũng biết rằng các biến đó thuộc về một khái niệm lớn hơn.

Hãy xem xét các phương thức trong _Listing 2-1_. Các biến có cần một ngữ cảnh có ý nghĩa hơn không? Tên hàm chỉ cung cấp một phần của ngữ cảnh, thuật toán cung cấp phần còn lại. Khi bạn đọc qua hàm, bạn thấy rằng ba biến, `number`, `verb` và `pluralModifier`, là một phần của thông báo "giả định thống kê". Thật không may, bối cảnh này phải suy ra mới có được. Khi bạn lần đầu xem xét phương thức, ý nghĩa của các biến là không rõ ràng.

**\# Listing 2-1: Biến với bối cảnh không rõ ràng**

```java
private void printGuessStatistics(char candidate, int count) {
    String number;
    String verb;
    String pluralModifier;
    if (count == 0) {
        number = "no";
        verb = "are";
        pluralModifier = "s";
    } else if (count == 1) {
        number = "1";
        verb = "is";
        pluralModifier = "";
    } else {
        number = Integer.toString(count);
        verb = "are";
        pluralModifier = "s";
    }
    String guessMessage = String.format("There %s %s %s%s", verb, number, candidate, pluralModifier);
    print(guessMessage);
}
```

Hàm này hơi dài và các biến được sử dụng xuyên suốt. Để tách hàm thành các phần nhỏ hơn, chúng ta cần tạo một lớp `GuessStatisticsMessage` và tạo ra ba biến của lớp này. Điều này cung cấp một bối cảnh rõ ràng cho ba biến. Chúng là một phần của `GuessStatisticsMessage`. Việc cải thiện bối cảnh cũng cho phép thuật toán được rõ ràng hơn bằng cách chia nhỏ nó thành nhiều chức năng nhỏ hơn. (Xem _Listing 2-2_.)

**\# Listing 2-2: Biến có ngữ cảnh**

```java
public class GuessStatisticsMessage {
    private String number;
    private String verb;
    private String pluralModifier;
    public String make(char candidate, int count) {
        createPluralDependentMessageParts(count);
        return String.format("There %s %s %s%s",verb, number, candidate, pluralModifier );
    }
    private void createPluralDependentMessageParts(int count) {
        if (count == 0) {
            thereAreNoLetters();
        } else if (count == 1) {
            thereIsOneLetter();
        } else {
            thereAreManyLetters(count);
        }
    }	
    private void thereAreManyLetters(int count) {
        number = Integer.toString(count);
        verb = "are";
        pluralModifier = "s";
    }
    private void thereIsOneLetter() {
        number = "1";
        verb = "is";
        pluralModifier = "";
    }
    private void thereAreNoLetters() {
        number = "no";
        verb = "are";
        pluralModifier = "s";
    }
}

```

Tên ngắn thường tốt hơn tên dài, miễn là chúng rõ ràng. Thêm đủ ngữ cảnh cho tên sẽ tốt hơn khi cần thiết.

Tên `accountAddress` và `customerAddress` là những tên đẹp cho trường hợp đặc biệt của lớp `Address` nhưng có thể là tên tồi cho các lớp khác. `Address` là một tên đẹp cho lớp. Nếu tôi cần phân biệt giữa địa chỉ MAC, địa chỉ cổng (port) và địa chỉ web thì tôi có thể xem xét `MAC`, `PostalAddress` và `URL`. Kết quả là tên chính xác hơn. Đó là tâm điểm của việc đặt tên.

## Lời kết

Điều khó khăn nhất của việc lựa chọn tên đẹp là nó đòi hỏi kỹ năng mô tả tốt và nền tảng văn hóa lớn. Đây là vấn đề về học hỏi hơn là vấn đề kỹ thuật, kinh doanh hoặc quản lý. Kết quả là nhiều người trong lĩnh vực này không học cách làm điều đó.

Mọi người cũng sợ đổi tên mọi thứ vì lo rằng người khác sẽ phản đối. Chúng tôi không chia sẻ nỗi sợ đó cho bạn. Chúng tôi thật sự biết ơn những ai đã đổi tên khác cho biến, hàm,…(theo hướng tốt hơn). Hầu hết thời gian chúng tôi không thật sự nhớ tên lớp và những phương thức của nó. Chúng tôi có các công cụ giúp chúng tôi trong việc đó để chúng tôi có thể tập trung vào việc code có dễ đọc hay không. Bạn có thể sẽ gây ngạc nhiên cho ai đó khi bạn đổi tên, giống như bạn có thể làm với bất kỳ cải tiến nào khác. Đừng để những cái tên tồi phá hủy sự nghiệp coder của mình.

Thực hiện theo một số quy tắc trên và xem liệu bạn có cải thiện được khả năng đọc code của mình hay không. Nếu bạn đang bảo trì code của người khác, hãy sử dụng các công cụ tái cấu trúc để giải quyết vấn đề này. Mất một ít thời gian nhưng có thể làm bạn nhẹ nhõm trong vài tháng.
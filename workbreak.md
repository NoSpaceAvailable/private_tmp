- Đầu tiên vào challenge thì ta thấy 1 trang login. Tuy nhiên do chưa có tài khoản nên ta đăng kí 1 cái:

![image](https://github.com/user-attachments/assets/0d64347c-b797-4f1c-ba5a-cfe58ed32734)

- Login vào tài khoản, ta thấy 1 trang web với vài thông tin người dùng kèm theo một cái performance chart, trông giống như là cái contribution chart của github:

![image](https://github.com/user-attachments/assets/ec003c2c-5a76-48cd-89e5-8ba9832d05dc)

- Ta có thể update một số field như là name, phone, position của account:

![image](https://github.com/user-attachments/assets/a28c225d-9d70-4eec-a7df-f3127fd2a510)

- Ngoài ra thì web còn có chức năng chat với support engineer. Bot yêu cầu send 1 URL, khi thử gửi cho bot 1 URL (không phải challenge) thì bot từ chối visit. Có vẻ đây là 1 challenge XSS:

![image](https://github.com/user-attachments/assets/f76699da-6580-4439-a57b-66e4cea0f13f)

- Challenge này không có source code, và các chức năng có vẻ cũng chỉ có vậy. F12 lên để xem source code HTML và JS ở phía frontend, sau một lúc nhìn code phía frontend thì mình nhanh chóng nhận ra
sink DOM-based XSS ở dòng 144 của profile.js:

![image](https://github.com/user-attachments/assets/956d7b57-771a-489e-bc9a-be4de339d873)

- Sink .innerHTML được để ở trong callback function của window.addEventListener. Tham số đầu tiên của addEventListener là 'message', tức là khi window nhận được một message thì nó sẽ thực thi callback
function. Trong lúc examine code thì mình có để ý là site thực hiện trao đổi message với iframe được nhúng bên trong nó ở khá nhiều nơi nên mình lần theo code để xem là chúng trao đổi gì với nhau.

- Để tiện thao tác hơn thì mình chuyển qua burpsuite.
- Tiếp tục đọc code js, ta nhận thấy rằng tại dòng thứ 52 có tạo một event listener cho performanceIframe. khi iframe này được load thì script thực hiện call method postMessage() đến iframe đó,
với parameter message được gửi đi là userTasks và targetOrigin là wildcard *. Về postMessage thì có thể đọc tại [đây](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage)

![image](https://github.com/user-attachments/assets/e93b3579-59da-4abd-8650-2db673cbf832)

- Nhìn lên dòng code thứ 51, ta thấy userTasks được lấy từ userSettings.tasks (hoặc là một array rỗng). Biến userSettings được lấy từ kết quả của method Object.assign. Method này thực hiện copy
toàn bộ các thuộc tính thỏa mãn các điều kiện như là enumerable và own property, đại khái là các thuộc tính này phải có thể được duyệt bằng vòng lặp và nó là thuộc tính của source object mà không phải
có được thông qua kế thừa:

![image](https://github.com/user-attachments/assets/1c860ca5-a700-4ec1-9b5f-504767d9a8e4)

![image](https://github.com/user-attachments/assets/892b3f1b-cc3d-4aa0-8d39-826c9e40b5ad)

- Nhìn kỹ hơn vào đoạn code thực hiện call Object.assign, ta thấy target object được lấy từ `profileData.assignedInfo`, `profileData` lại là kết quả khi thực hiện call API fetch đến `/api/user/profile/${userId}`.

```javascript
...
        const response = await fetch(`/api/user/profile/${userId}`);
        const profileData = await response.json();
        if (response.ok) {
            const userSettings = Object.assign(
                { name: "", phone: "", position: "" },
                profileData.assignedInfo
            );
...
```

- Khi trace request ở endpoint này trong burpsuite thì mình nhận ra là `assignedInfo` trả về chỉ chứa các key là name, phone, position mà không có tasks. Vậy thì, kết quả của đoạn code `userTasks = userSettings.tasks || [];`
đã nhắc ở phía trên sẽ luôn là `userTasks == []`:

![image](https://github.com/user-attachments/assets/fa2939c8-4291-4932-a7bd-2b2752173d86)

- Đến đây thì mình tự hỏi là có cách nào initialize thuộc tính `tasks` cho `assignedInfo` mà server trả về không? Quay lại với việc trace log burpsuite thì mình thấy tại endpoint `/api/user/settings` có thực hiện
POST một đoan JSON lên server gồm key name, position và phone:

![image](https://github.com/user-attachments/assets/3f659489-0d0e-4d65-a668-52d497ed7253)

- Vì bài này blackbox nên mình không thể biết được backend xử lí đoạn JSON trên như nào. Mình thử đưa request trên vào repeater và POST lên server kèm theo key tasks xem sao. Tuy nhiên đời không như là
mơ, backend trả về response như sau:

![image](https://github.com/user-attachments/assets/a0e07fc6-5f9c-4a77-9161-1a5da400b27c)

- Lúc này, dựa trên kinh nghiệm của mình thì mình đoán là ở chỗ này có thể backend bị lổ hổng prototype pollution, nên mình thực hiện lại request với JSON như sau:

![image](https://github.com/user-attachments/assets/1c0738e8-a66f-4e81-a000-600c8ddc6efa)

- Mình thử request đến endpoint `/api/user/profile/${id}` để kiểm tra `assignedInfo`. Có thể thấy là `assignedInfo` đã thay đổi:

![image](https://github.com/user-attachments/assets/34cb3239-7e59-462f-a9dc-d00b06481aaa)

- Sau khi xin được source của challenge thì thực chất ở phía backend bị lỗ hổng mass assignment tại function settings

![image](https://github.com/user-attachments/assets/a5e1421d-6ccc-4bdd-8d09-0a4788eb427e)

- Quay lại với profile.js, ta thấy rằng trước khi `profileData.assignedInfo` được đưa vào làm target cho Object.assign thì method .json() được gọi lên để parse response và lưu vào biến profileData. 
Thử test ở dev console của browser thì ta thu được kết quả như sau:

![image](https://github.com/user-attachments/assets/f163373c-7714-4e94-a632-8715ef1d32db)

- Có thể thấy được là result object của method Object.assign lúc này đã có thuộc tính tasks. Vậy thì ta có thể thay đổi tasks một cách tùy ý, tuy nhiên cần thay đổi tasks thành cái gì? Lúc này mình chợt
nhớ ra là site có thực hiện gửi userTasks dưới dạng message đến performanceIframe bằng .postMessage() nên mình lần qua mã js của iframe để đọc.

- Ở mã js phía iframe cũng có register một event listener khi có message được gửi đến. Data của event được đưa vào hàm renderPerformanceChart():

![image](https://github.com/user-attachments/assets/387adf04-0ffc-4156-84e7-b4aa548921ff)

![image](https://github.com/user-attachments/assets/ca1129ff-6157-4aec-9cda-1aaae04080c5)

- Trace vào hàm này thì mình thấy nó làm khá nhiều thứ linh tinh nên mình chỉ quan tâm xem parameter truyền vào được đưa đi đâu. Ở đây mình thấy là parameter `taskData` được đưa tiếp vào hàm
generateTaskHeatmapData():

![image](https://github.com/user-attachments/assets/44db2a2e-dccc-4ff4-a74e-0ce5cfefcbf8)

![image](https://github.com/user-attachments/assets/354ebfa7-eb00-4b8c-9ef2-280683709f03)

- Ở dòng code thứ 106, ta thấy method taskData.map được gọi lên. Trong JS thì .map là method dùng để chuyển một array thành một array mới => taskData nên là array. Ở đây thì method này duyệt qua
taskData, mỗi element được duyệt thì đều được lấy ra thuộc tính `date` và `tasksCompleted` => taskData là một array chứa các object có thuộc tính date và tasksCompleted. Vòng for thì format ngày
tháng năm lại cho đẹp, đọc thấy cũng không quan trọng lắm nên có thể bỏ qua.

- Quay lại đoạn code ở phía trên thì `tasksCompleted` được render thẳng vào DOM thông qua method .html(). Theo như mình search google thì method này thực hiện set innerHTML content của một HTML
element. Mình thử test inject số 99 vào `tasksCompleted` như sau:

![image](https://github.com/user-attachments/assets/1d155f74-dc19-4ab2-8da4-e60039d29358)

- Vì lí do nào đó mà tasksCompleted không được update lên 99. Mình thử debug thì thấy có đoạn code sau:

![image](https://github.com/user-attachments/assets/21710135-4e5c-42cb-96fa-cbb22e9a465b)

- Sửa date thành 2024-12-05 và send lại số 99:

![image](https://github.com/user-attachments/assets/d34792b5-a472-45e0-9092-ac296ed7546b)

- Vì tasksCompleted không bị restrict gì cả nên thử sửa nó thành 1 payload XSS đơn giản:

![image](https://github.com/user-attachments/assets/513129c4-55fc-42c8-a651-a656ecf366b1)

- Tiến hành update user setting:

![image](https://github.com/user-attachments/assets/2172c052-7a35-49c1-9cb9-a214e997a584)

- Tuy nhiên khi access vào trang profile thì không có popup alert hiện lên. Lí do là vì iframe đã bị sandbox như sau:

![image](https://github.com/user-attachments/assets/c707921d-d76a-40c0-bd1f-b614c1dff30c)

![image](https://github.com/user-attachments/assets/ded12190-d2ac-4091-bce2-714694df3447)

- Nếu access vào document.cookie thì sẽ bị lỗi sau:

![image](https://github.com/user-attachments/assets/1520f94f-d103-4c54-973f-aaca290bbaae)

- Message này thông báo rằng iframe bị sandbox và không có flag allow-same-origin. Flag này có ý nghĩa là:

![image](https://github.com/user-attachments/assets/65d9af1f-8296-4a08-80d6-ab6b8e253798)

- Đến đây thì để tìm hiểu nguyên nhân, mình thử cho iframe fetch đến webhook như sau:

![image](https://github.com/user-attachments/assets/38a31bf2-b94b-4f96-bd7e-bb0157c6986f)

![image](https://github.com/user-attachments/assets/a093ccac-b0b0-43cb-9267-d65964394fb6)

- Sandbox này hiện chưa không cách để bypass. Để có thể send được payload XSS đến bot thì ta cần tìm cách khác. Nếu mọi người chưa quên thì sink innerHTML ở đầu writeup chưa được sử dụng đến:

![image](https://github.com/user-attachments/assets/494a7866-ab0b-4e28-8645-fdfafc72b51d)

- Ở đây, khi có một message được gửi tới thì code thực hiện lấy thuộc tính totalTasks rồi đưa thẳng vào innerHTML như trên hình. Vậy thì, thử cho iframe post một message ngược lại parent window:

![image](https://github.com/user-attachments/assets/c2f46a70-b56c-4e32-9d55-aea22a5beef5)

![image](https://github.com/user-attachments/assets/3a6585a6-a983-450d-8f2c-fb2a20d7702a)

- Sửa totalTasks thành payload:

```json
{
  "name": "xxxxxxxxxxxxxx",
  "phone": "9999999999",
  "position": "xxxxxxxxxxxxxx",
  "__proto__": {
    "tasks": [
      {
        "date": "2024-12-05",
        "tasksCompleted": "<img src=x onerror=eval(atob('cGFyZW50LnBvc3RNZXNzYWdlKHsndG90YWxUYXNrcyc6JzxpbWcvc3JjL29uZXJyb3I9YWxlcnQob3JpZ2luKT4nfSwnKicp'))>"
      }
    ]
  }
}
```

![image](https://github.com/user-attachments/assets/353a7ef1-ebcd-413a-a107-05b899a08f1d)

- Kết quả:

![image](https://github.com/user-attachments/assets/391b872a-4f77-44e1-95c7-7942b85044e4)

- Giờ chỉ cần lấy cookie của admin bot là xong bài.

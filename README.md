# blockchain
pragma solidity ^0.4.24;
import "./DateTime.sol";
import "./HouseData.sol";
import "./ConvertETH_USD.sol";
contract HouseRenting is HousesData{
    
    DateTime datetime = new DateTime(); // Khởi tạp đối tượng Dateime để sử dụng.
    
  
    struct House_Contract {
        uint id_nha; // ID ngôi nhà renter chọn
        address renter; // địa chỉ người thuê. Default: người tạo hợp đồng = người thuê.
        uint _giaNha; // Giá căn nhà - tính đơn vị $ trên / Tháng
        uint _thoiGianBatDau; // Thời gian bắt đầu.
        uint _thoiGianKetThuc; // Thời gian kết thúc. 
        uint _soluongThang; // Tổng số lượng tháng của người thuê.
        uint countMonth; // Đếm số lượng tháng đã thanh toán. Default contructor: countMonth = 0.
        string email; // Địa chỉ emal.
        bool status; // Trạng thái của hợp đồng: True == Còn sử dụng và ngược lại.
    }
  
    // kiểm tra xem có phải là người tạo contract/house hay không ?
    modifier onlyOwner() {
        if(msg.sender != owner){
            revert();
        }
    _;
    }
    // Kiểm tra trạng thái ngôi nhà có đang ở hay không.
    modifier checkStatus(uint idHouse) {
        if(houseStructs[idHouse].status != false){
            revert();
        }
    _;
    }
    // Kiểm tra có phải người thuê không ?
    modifier onlyRenter() {
        if(msg.sender != houseContract[msg.sender].renter){
            revert();
        }
    _;
    }
    
    mapping (address => House_Contract) public houseContract; // Mapping theo địa chỉ người thuê -> hợp đồng của người đó.
    address[] public contractList;
    uint public countContract; // Đếm tổng số lượng hợp đồng.
    mapping (uint => address[]) public userList; // Mapping danh sách người dùng theo kiểu house[index][index người thuê]. Trả về địa chỉ người thuê.
    
    // Xem lịch sử thuê nhà với tham số truyền vào như kiểu mapping userList.
    function getHouseHistory(uint idHouse, 
                             uint indexUser) 
                             public view returns (uint,address[]) {
        require(indexUser <= countContract);
        return(idHouse,userList[indexUser]);
    }
    // Khởi tạo hợp đồng thuê nhà.
    function newHouseContract(uint id_house,
                              uint16 yearStart, 
                              uint8 monthStart, 
                              uint16 yearEnd, 
                              uint8 monthEnd,
                              string email)
                              payable public returns(uint rowNumber) {
        require(houseStructs[id_house].status == false); // Điều kiện đc tạo khi trạng thái ngôi nhà đó == False == không có người nào thuê.
        houseContract[msg.sender].id_nha = id_house;
    
        houseContract[msg.sender].renter = msg.sender;
        houseContract[msg.sender].email=email;
        houseContract[msg.sender]._giaNha = houseStructs[id_house].price;
        houseContract[msg.sender]._thoiGianBatDau = datetime.toTimestamp(yearStart,monthStart,1,0,0,0); 
        houseContract[msg.sender]._thoiGianKetThuc =  datetime.toTimestamp(yearEnd,monthEnd,1,0,0,0);
        if(datetime.getYear(houseContract[msg.sender]._thoiGianKetThuc) > datetime.getYear(houseContract[msg.sender]._thoiGianBatDau)) {
            uint getMonthByYears = (datetime.getYear(houseContract[msg.sender]._thoiGianKetThuc) - datetime.getYear(houseContract[msg.sender]._thoiGianBatDau)) *12;
            houseContract[msg.sender]._soluongThang = datetime.getMonth(houseContract[msg.sender]._thoiGianKetThuc) - datetime.getMonth(houseContract[msg.sender]._thoiGianBatDau) + getMonthByYears +1;
        } else houseContract[msg.sender]._soluongThang = datetime.getMonth(houseContract[msg.sender]._thoiGianKetThuc) - datetime.getMonth(houseContract[msg.sender]._thoiGianBatDau) +1;
        houseContract[msg.sender].countMonth = 0;
        
        // Sau khi xong điền xong tất cả thông tin thì người thuê gửi tiền đặt cọc vào contract.
        sendDeposit(houseStructs[id_house].price);
        // Thêm người thuê vào userList.
        userList[id_house].push(msg.sender);
        houseStructs[id_house].status=true;
        houseContract[msg.sender].status=true;
        countContract++;
        return contractList.push(msg.sender) - 1;
        
    }
    // Hàm gửi tiền đặt cọc vào contract. Yêu cầu số tiền gửi = giá nhà 1 tháng.
    function sendDeposit(uint256 amount) onlyRenter payable public{
        require(msg.value == amount);
    }
    
    // Trả Lại tiền đặt cọc cho người thuê.
    // Trường hợp chưa hết thời gian hợp đồng thì số tiền đặt cọc sẽ gửi cho người chủ => Mất tiền đặt cọc.
    // Trường hợp nếu đã đủ số lượng tháng thì số tiền từ balance trở về ví của người thuê.
    function refundDesposit() public onlyRenter{
        if (houseContract[msg.sender].countMonth == houseContract[msg.sender]._soluongThang) {
            (msg.sender).transfer( houseStructs[houseContract[msg.sender].id_nha].price);
        } else {
            owner.transfer(address(this).balance);
        }
        houseStructs[houseContract[msg.sender].id_nha].status=false;
         houseContract[msg.sender].status=false;
    }
    
    // Hàm trả tiền nhà mỗi tháng.
    function payEachMonth() onlyRenter payable public {
        
        uint amount;
        // kiểm tra điều kiện số lượng tháng đã thanh toán < số lượng tháng ? true
        if(houseContract[msg.sender].countMonth < houseContract[msg.sender]._soluongThang ){ 
            amount = houseContract[msg.sender]._giaNha;
        }
        else revert();
        require(msg.value == amount);
        uint8 month = datetime.getMonth(houseContract[msg.sender]._thoiGianBatDau) +1;
        uint16 year;
        if(month > 12){
            year = datetime.getYear(houseContract[msg.sender]._thoiGianBatDau) +1;
            month = 1;
        }
        else  year = datetime.getYear(houseContract[msg.sender]._thoiGianBatDau);
        owner.transfer(amount);
        houseContract[msg.sender]._thoiGianBatDau = datetime.toTimestamp(year,month,1,0,0,0);
        houseContract[msg.sender].countMonth++;
    }
    // Xem số lượng tiền hiện tại mà contract đang giữ.
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
    
    function() public payable {
    }
    

   
   
  
}

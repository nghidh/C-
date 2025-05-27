# Dự án Hệ thống đăng nhập và đăng ký tài khoản và quản lý ví điểm thưởng (C++)

1. Giới thiệu Dự án
Hệ thống quản lý ví điểm là giải pháp tích hợp:
Đăng ký/Đăng nhập với xác thực 2 lớp OTP
Quản lý ví điện tử với chức năng chuyển điểm
Theo dõi lịch sử giao dịch
Phân quyền người dùng (thường/quản trị)
---

2. Thành viên Nhóm & Phân công
   
Tên thành viên | Mã Sinh Viên |	Vai trò	Công việc chính

Đỗ Hữu Nghị    | K24DTCN528   |	Trưởng nhóm	Thiết kế hệ thống, Quản lý giao diện, 	Xử lý nghiệp vụ ví, Giao dịch, Triển khai OTP, Hashing

---
3. Đặc tả chức năng

Module 1: Quản lý Người dùng
Đăng ký tài khoản mới
Xác thực đăng nhập
Thay đổi mật khẩu
Khóa/mở tài khoản (admin)

Module 2: Quản lý Ví
Tạo ví mới
Chuyển điểm giữa các ví
Xem lịch sử giao dịch
Kiểm tra số dư

Module 3: Bảo mật
Mã hóa SHA256
OTP 6 số
Sao lưu dữ liệu

---
4. Cách tải chương trình: https://github.com/nghidh/C-

---
5. cách chạy chương trình, kèm các thao tác thực hiện
5.1 Đăng ký tài khoản

  - Chọn "Đăng ký" từ menu
  - Nhập tên người dùng: user123
  - Nhập mật khẩu (hoặc để trống để tự sinh): 
  - Hệ thống hiển thị mật khẩu mới (nếu có)
  - Đăng nhập với thông tin nhận được

5.2 Chuyển điểm

  - Đăng nhập thành công
  - Chọn "Chuyển điểm"
  - Nhập ví đích: WALLET_456
  - Nhập số điểm: 100
  - Nhập OTP nhận được: 123456
  - Kiểm tra kết quả giao dịch

---
## MÃ NGUỒN C++**

#include <iostream>
#include <fstream>
#include <vector>
#include <ctime>
#include <sstream>
#include <iomanip>
#include <openssl/sha.h>
#include <algorithm>

using namespace std;

// Utilities
string sha256(const string &str) {
    unsigned char hash[SHA256_DIGEST_LENGTH];
    SHA256_CTX sha256;
    SHA256_Init(&sha256);
    SHA256_Update(&sha256, str.c_str(), str.size());
    SHA256_Final(hash, &sha256);

    stringstream ss;
    for(int i = 0; i < SHA256_DIGEST_LENGTH; i++) {
        ss << hex << setw(2) << setfill('0') << (int)hash[i];
    }
    return ss.str();
}

string generateOTP() {
    srand(time(NULL));
    return to_string(100000 + rand() % 900000);
}

// Transaction class
class Transaction {
public:
    string from;
    string to;
    int amount;
    time_t timestamp;
    string status;

    Transaction(string f, string t, int a, string s) 
        : from(f), to(t), amount(a), status(s), timestamp(time(0)) {}
};

// Wallet class
class Wallet {
private:
    string id;
    int balance;
    vector<Transaction> history;

public:
    Wallet(string walletId) : id(walletId), balance(0) {}

    string getId() const { return id; }
    int getBalance() const { return balance; }
    
    bool transferTo(Wallet &target, int amount, string otp) {
        if (balance < amount) return false;
        
        balance -= amount;
        target.balance += amount;
        
        history.emplace_back(id, target.id, amount, "SUCCESS");
        target.history.emplace_back(id, target.id, amount, "SUCCESS");
        return true;
    }

    void printHistory() const {
        cout << "Transaction history for wallet " << id << ":\n";
        for (const auto &tx : history) {
            cout << ctime(&tx.timestamp) 
                 << tx.from << " -> " << tx.to 
                 << " Amount: " << tx.amount 
                 << " Status: " << tx.status << endl;
        }
    }
};

// User class
class User {
private:
    string username;
    string passwordHash;
    string role;
    bool tempPassword;
    Wallet wallet;

public:
    User(string uname, string pwdHash, string r, string walletId) 
        : username(uname), passwordHash(pwdHash), role(r), 
          tempPassword(true), wallet(walletId) {}

    string getUsername() const { return username; }
    string getRole() const { return role; }
    Wallet& getWallet() { return wallet; }

    bool verifyPassword(const string &pwd) const {
        return sha256(pwd) == passwordHash;
    }

    void changePassword(const string &newPwd) {
        passwordHash = sha256(newPwd);
        tempPassword = false;
    }

    bool isTempPassword() const { return tempPassword; }
};

// Database handler
class Database {
private:
    static const string USER_FILE;
    static const string BACKUP_FILE;

public:
    static vector<User> loadUsers() {
        vector<User> users;
        ifstream file(USER_FILE);
        // Implementation to read users from file
        return users;
    }

    static void saveUsers(const vector<User> &users) {
        ofstream file(USER_FILE);
        // Implementation to save users to file
    }

    static void backup() {
        time_t now = time(0);
        string backupName = "backup_" + to_string(now) + ".dat";
        ifstream src(USER_FILE, ios::binary);
        ofstream dst(backupName, ios::binary);
        dst << src.rdbuf();
    }
};

// Authentication system
class AuthSystem {
private:
    vector<User> users;

public:
    bool login(const string &username, const string &password) {
        auto it = find_if(users.begin(), users.end(), 
            [&](const User &u) { return u.getUsername() == username; });
        
        if (it != users.end() && it->verifyPassword(password)) {
            if (it->isTempPassword()) {
                cout << "You must change your temporary password!\n";
                string newPwd;
                cout << "Enter new password: ";
                cin >> newPwd;
                it->changePassword(newPwd);
                Database::saveUsers(users);
            }
            return true;
        }
        return false;
    }

    void createAccount(bool isAdmin) {
        string username, password;
        cout << "Enter username: ";
        cin >> username;
        cout << "Enter password (empty for auto-generate): ";
        cin >> password;

        if (password.empty()) {
            password = generateOTP();
            cout << "Generated password: " << password << endl;
        }

        string role = isAdmin ? "admin" : "user";
        users.emplace_back(username, sha256(password), role, "WALLET_" + username);
        Database::saveUsers(users);
    }
};

// Transaction system
class WalletSystem {
public:
    static bool transferFunds(User &from, User &to, int amount) {
        string otp = generateOTP();
        cout << "OTP sent to user: " << otp << endl;
        
        string inputOtp;
        cout << "Enter OTP to confirm: ";
        cin >> inputOtp;
        
        if (inputOtp != otp) {
            cout << "Invalid OTP\n";
            return false;
        }

        return from.getWallet().transferTo(to.getWallet(), amount, otp);
    }
};

// Main menu
void userMenu(User &user) {
    // Implementation for user functions
}

void adminMenu(User &admin) {
    // Implementation for admin functions
}

int main() {
    AuthSystem auth;
    int choice;
    
    while(true) {
        cout << "1. Login\n2. Register\n3. Exit\nChoice: ";
        cin >> choice;
        
        if (choice == 1) {
            string username, password;
            cout << "Username: "; cin >> username;
            cout << "Password: "; cin >> password;
            
            if (auth.login(username, password)) {
                // Load user and show menu
            } else {
                cout << "Login failed!\n";
            }
        } else if (choice == 2) {
            auth.createAccount(false);
        } else {
            break;
        }
    }
    
    return 0;
}

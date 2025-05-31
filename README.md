#include <iostream>
using namespace std;

struct Pelaku {
    string nama;
    string jabatan;
    string kerugian; 
    Pelaku* left;
    Pelaku* right;
    Pelaku(string n, string j, string k) : nama(n), jabatan(j), kerugian(k), left(nullptr), right(nullptr) {}
};

struct LogNode {
    string aksi;
    string nama;
    LogNode* next;
    LogNode(string a, string n) : aksi(a), nama(n), next(nullptr) {}
};

enum AksiType { TAMBAH, HAPUS };
struct AksiUndo {
    AksiType tipe;
    Pelaku data;
    AksiUndo* next;
    AksiUndo(AksiType t, const Pelaku& d) : tipe(t), data(d), next(nullptr) {}
};

struct QueueNode {
    string nama;
    QueueNode* next;
    QueueNode(string n) : nama(n), next(nullptr) {}
};

Pelaku* root = nullptr;
AksiUndo* undoTop = nullptr; // Stack manual
LogNode* logHead = nullptr;  // Linked list log
QueueNode* qFront = nullptr; // Queue manual
QueueNode* qBack = nullptr;

void pushUndo(AksiType tipe, const Pelaku& data) {
    AksiUndo* baru = new AksiUndo(tipe, data);
    baru->next = undoTop;
    undoTop = baru;
}
bool stackUndoEmpty() {
    return undoTop == nullptr;
}
AksiUndo popUndo() {
    if (!undoTop) throw runtime_error("Stack kosong");
    AksiUndo temp = *undoTop;
    AksiUndo* del = undoTop;
    undoTop = undoTop->next;
    delete del;
    return temp;
}

void enqueue(const string& nama) {
    QueueNode* baru = new QueueNode(nama);
    if (!qFront) {
        qFront = qBack = baru;
    } else {
        qBack->next = baru;
        qBack = baru;
    }
}
bool queueEmpty() {
    return qFront == nullptr;
}
void dequeue() {
    if (!qFront) {
        cout << "Tidak ada pelaku dalam antrian pengadilan.\n";
        return;
    }
    cout << "Memproses pelaku: " << qFront->nama << endl;
    QueueNode* del = qFront;
    qFront = qFront->next;
    if (!qFront) qBack = nullptr;
    delete del;
}
void tampilkanAntrian() {
    if (!qFront) {
        cout << "Tidak ada pelaku dalam antrian pengadilan.\n";
        return;
    }
    cout << "Daftar Antrian Pengadilan:\n";
    QueueNode* temp = qFront;
    while (temp) {
        cout << "- " << temp->nama << endl;
        temp = temp->next;
    }
}

Pelaku* buatNode(string nama, string jabatan, string kerugian) {
    return new Pelaku(nama, jabatan, kerugian);
}

Pelaku* tambahPelaku(Pelaku* node, string nama, string jabatan, string kerugian, bool logUndo = true) {
    if (!node) {
        if (logUndo) pushUndo(TAMBAH, Pelaku(nama, jabatan, kerugian));
        LogNode* baru = new LogNode("Tambah", nama);
        baru->next = logHead;
        logHead = baru;
        cout << "Pelaku bernama " << nama << " berhasil ditambahkan.\n";
        return buatNode(nama, jabatan, kerugian);
    }
    if (nama < node->nama)
        node->left = tambahPelaku(node->left, nama, jabatan, kerugian, logUndo);
    else if (nama > node->nama)
        node->right = tambahPelaku(node->right, nama, jabatan, kerugian, logUndo);
    else
        cout << "Nama pelaku sudah ada, tidak bisa ditambahkan lagi.\n";
    return node;
}

void inorder(Pelaku* node) {
    if (node) {
        inorder(node->left);
        cout << "- " << node->nama << " | Jabatan: " << node->jabatan << " | Kerugian: Rp" << node->kerugian << endl;
        inorder(node->right);
    }
}

Pelaku* cariMin(Pelaku* node) {
    while (node && node->left) node = node->left;
    return node;
}

Pelaku* hapusPelaku(Pelaku* node, string nama, bool logUndo = true, Pelaku* &hapusData = *(Pelaku**)nullptr) {
    if (!node) {
        cout << "Data tidak ditemukan.\n";
        return nullptr;
    }
    if (nama < node->nama)
        node->left = hapusPelaku(node->left, nama, logUndo, hapusData);
    else if (nama > node->nama)
        node->right = hapusPelaku(node->right, nama, logUndo, hapusData);
    else {
        if (logUndo) pushUndo(HAPUS, *node);
        // Tambah ke log linked list
        LogNode* baru = new LogNode("Hapus", nama);
        baru->next = logHead;
        logHead = baru;
        hapusData = node;
        if (!node->left) {
            Pelaku* temp = node->right;
            delete node;
            return temp;
        } else if (!node->right) {
            Pelaku* temp = node->left;
            delete node;
            return temp;
        } else {
            Pelaku* temp = cariMin(node->right);
            node->nama = temp->nama;
            node->jabatan = temp->jabatan;
            node->kerugian = temp->kerugian;
            node->right = hapusPelaku(node->right, temp->nama, false, hapusData);
        }
    }
    return node;
}

Pelaku* undo(Pelaku* node) {
    if (stackUndoEmpty()) {
        cout << "Tidak ada aksi yang bisa di-undo.\n";
        return node;
    }
    AksiUndo aksi = popUndo();

    if (aksi.tipe == TAMBAH) {
        cout << "Undo tambah: " << aksi.data.nama << endl;
        Pelaku* dummy = nullptr;
        node = hapusPelaku(node, aksi.data.nama, false, dummy);
        LogNode* baru = new LogNode("UndoTambah", aksi.data.nama);
        baru->next = logHead;
        logHead = baru;
    } else if (aksi.tipe == HAPUS) {
        cout << "Undo hapus: " << aksi.data.nama << endl;
        node = tambahPelaku(node, aksi.data.nama, aksi.data.jabatan, aksi.data.kerugian, false);
        LogNode* baru = new LogNode("UndoHapus", aksi.data.nama);
        baru->next = logHead;
        logHead = baru;
    }
    return node;
}

void tampilkanLog() {
    if (!logHead) {
        cout << "Belum ada log aksi.\n";
        return;
    }
    cout << "Riwayat Aksi (terbaru ke terlama):\n";
    LogNode* temp = logHead;
    while (temp) {
        cout << "- " << temp->aksi << ": " << temp->nama << endl;
        temp = temp->next;
    }
}

int main() {
    int pilihan;
    string nama, jabatan, kerugian;

    do {
        cout << "\nMenu:\n";
        cout << "1. Tambah Pelaku Korupsi\n";
        cout << "2. Tampilkan Daftar Pelaku (A-Z)\n";
        cout << "3. Hapus Pelaku Korupsi\n";
        cout << "4. Undo Aksi Terakhir\n";
        cout << "5. Tampilkan Log Aksi (Linked List)\n";
        cout << "6. Tambah ke Antrian Pengadilan (Queue)\n";
        cout << "7. Proses Antrian Pengadilan (Queue)\n";
        cout << "8. Tampilkan Antrian Pengadilan (Queue)\n";
        cout << "0. Keluar\n";
        cout << "Pilih menu: ";
        cin >> pilihan;
        cin.ignore();

        switch (pilihan) {
            case 1:
                cout << "Masukkan nama pelaku: ";
                getline(cin, nama);
                cout << "Masukkan jabatan saat korupsi: ";
                getline(cin, jabatan);
                cout << "Masukkan nominal kerugian negara (rupiah): ";
                getline(cin, kerugian);
                root = tambahPelaku(root, nama, jabatan, kerugian);
                break;
            case 2:
                cout << "\nDaftar Pelaku Korupsi (A-Z):\n";
                inorder(root);
                break;
            case 3:
                cout << "Masukkan nama pelaku yang ingin dihapus: ";
                getline(cin, nama);
                {
                    Pelaku* dummy = nullptr;
                    root = hapusPelaku(root, nama, true, dummy);
                }
                break;
            case 4:
                root = undo(root);
                break;
            case 5:
                tampilkanLog();
                break;
            case 6:
                cout << "Masukkan nama pelaku yang akan masuk antrian pengadilan: ";
                getline(cin, nama);
                enqueue(nama);
                cout << "Pelaku " << nama << " masuk ke antrian pengadilan.\n";
                break;
            case 7:
                dequeue();
                break;
            case 8:
                tampilkanAntrian();
                break;
            case 0:
                cout << "Keluar dari program.\n";
                break;
            default:
                cout << "Pilihan tidak valid.\n";
        }
    } while (pilihan != 0);

    while (logHead) {
        LogNode* temp = logHead;
        logHead = logHead->next;
        delete temp;
    }
    while (undoTop) {
        AksiUndo* temp = undoTop;
        undoTop = undoTop->next;
        delete temp;
    }
    while (qFront) {
        QueueNode* temp = qFront;
        qFront = qFront->next;
        delete temp;
    }

    return 0;
}

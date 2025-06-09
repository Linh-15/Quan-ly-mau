<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Ứng dụng Quản lý Mẫu Đèn Showroom</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <!-- Chosen Palette: Calm Harmony (Slate/Gray background, Muted Blue for actions, Green/Orange/Yellow for status) -->
    <!-- Application Structure Plan: A two-tab SPA. Tab 1 is a Dashboard & Management view showing current inventory status via a donut chart, key stats, an interactive table for loan/return actions, and a dedicated table for currently loaned items. Tab 2 shows a complete, searchable loan history. This structure separates the 'live' operational view from the 'archival' historical log, which is a more intuitive workflow than Excel sheets. Modals are used for data entry tasks (loaning, returning, adding items) to guide the user and prevent errors. -->
    <!-- Visualization & Content Choices: 1. Inventory Status (Available vs. Loaned): Goal=Inform, Viz=Donut Chart, Interaction=Hover, Justification=At-a-glance overview, Lib=Chart.js/Canvas. 2. Sample List: Goal=Manage, Viz=Interactive HTML Table, Interaction=Search/Sort/Buttons, Justification=Core management interface, Lib=HTML+JS. 3. Loan/Return/Add Forms: Goal=Task Execution, Viz=Modal Forms, Interaction=User Input, Justification=Guided data entry, Lib=HTML+JS. 4. Loan History: Goal=Audit, Viz=Interactive HTML Table, Interaction=Search/Sort, Justification=Easy historical lookup, Lib=HTML+JS. 5. Product Description Generator: Goal=Enhance Data Entry, Viz=Text Area, Interaction=Button click to generate with LLM, Justification=Automate content creation for new items, Lib=Gemini API (gemini-2.0-flash). 6. Sample Image Display: Goal=Visual Identification, Viz=Image, Interaction=None, Justification=Quick visual lookup for items, Lib=HTML (img tag). 7. Project Tracking: Goal=Categorize Loans, Viz=Input Field in Modal, Column in History Table, Interaction=User Input, Justification=Associate loans with specific projects for better tracking and reporting. 8. Loaned Samples Table: Goal=Track Outstanding Loans, Viz=Filtered HTML Table, Interaction=Directly from Dashboard, Justification=Provides a quick overview of currently unreturned items for follow-up. 9. Edit Sample Form: Goal=Update Item Details, Viz=Modal Form, Interaction=User Input to modify existing data fields (name, location, image, description, and now `maDen`), Justification=Allows full data management for existing items, reducing need for re-entry and ensuring data consistency. -->
    <!-- CONFIRMATION: NO SVG graphics used. NO Mermaid JS used. -->
    <style>
        body {
            font-family: 'Inter', sans-serif;
        }
        .tab-button.active {
            border-bottom-color: #3b82f6;
            color: #3b82f6;
            font-weight: 600;
        }
        .modal-backdrop {
            background-color: rgba(0,0,0,0.5);
            transition: opacity 0.3s ease;
        }
        .loading-spinner {
            border: 4px solid #f3f3f3;
            border-top: 4px solid #3498db;
            border-radius: 50%;
            width: 20px;
            height: 20px;
            animation: spin 1s linear infinite;
        }
        @keyframes spin {
            0% { transform: rotate(0deg); }
            100% { transform: rotate(360deg); }
        }
    </style>
</head>
<body class="bg-slate-50_text-slate-800">

    <div id="app" class="max-w-7xl mx-auto p-4 sm:p-6 lg:p-8">
        <header class="mb-8">
            <h1 class="text-3xl font-bold text-slate-900">Trình Quản Lý Mẫu Đèn Showroom</h1>
            <p class="text-slate-600 mt-1">Công cụ giúp bạn theo dõi và quản lý các mẫu đèn một cách hiệu quả.</p>
        </header>

        <div class="border-b border-slate-200 mb-6">
            <nav class="-mb-px flex space-x-6" aria-label="Tabs">
                <button id="tab-dashboard" class="tab-button active whitespace-nowrap py-4 px-1 border-b-2 font-medium text-sm border-transparent text-slate-500 hover:text-slate-700 hover:border-slate-300">
                    Bảng Điều Khiển & Quản Lý
                </button>
                <button id="tab-history" class="tab-button whitespace-nowrap py-4 px-1 border-b-2 font-medium text-sm border-transparent text-slate-500 hover:text-slate-700 hover:border-slate-300">
                    Lịch Sử Mượn Trả
                </button>
            </nav>
        </div>

        <main>
            <div id="content-dashboard" class="tab-content">
                <section id="dashboard-stats" class="mb-8">
                     <div class="mb-6">
                        <h2 class="text-xl font-semibold text-slate-900">Tổng Quan</h2>
                        <p class="text-slate-500 mt-1">Thống kê nhanh về tình trạng các mẫu đèn trong showroom.</p>
                    </div>
                    <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
                        <div class="bg-white p-6 rounded-lg shadow-sm flex flex-col justify-between">
                            <div>
                                <h3 class="text-slate-500 font-medium">Tổng số mẫu đèn</h3>
                                <p id="total-samples" class="text-3xl font-bold text-blue-600 mt-2">0</p>
                            </div>
                        </div>
                        <div class="bg-white p-6 rounded-lg shadow-sm flex flex-col justify-between">
                            <div>
                                <h3 class="text-slate-500 font-medium">Đang cho mượn</h3>
                                <p id="loaned-samples" class="text-3xl font-bold text-amber-600 mt-2">0</p>
                            </div>
                        </div>
                        <div class="bg-white p-6 rounded-lg shadow-sm flex flex-col justify-between">
                             <div>
                                <h3 class="text-slate-500 font-medium">Sẵn sàng</h3>
                                <p id="available-samples" class="text-3xl font-bold text-green-600 mt-2">0</p>
                            </div>
                        </div>
                         <div class="md:col-span-3 bg-white p-6 rounded-lg shadow-sm">
                            <h3 class="text-slate-500 font-medium mb-4">Tỷ lệ trạng thái mẫu</h3>
                            <div class="chart-container" style="position: relative; height:250px; width:100%; max-width: 400px; margin: auto;">
                                <canvas id="statusChart"></canvas>
                            </div>
                        </div>
                    </div>
                </section>

                <section id="outstanding-loans" class="mb-8">
                    <div class="mb-6">
                        <h2 class="text-xl font-semibold text-slate-900">Mẫu Đèn Đang Cho Mượn</h2>
                        <p class="text-slate-500 mt-1">Bảng này hiển thị tất cả các mẫu đèn hiện đang được mượn, giúp bạn dễ dàng theo dõi và nhắc nhở khi cần.</p>
                    </div>
                    <div class="overflow-x-auto bg-white rounded-lg shadow-sm">
                        <table class="w-full text-sm text-left text-slate-500">
                            <thead class="text-xs text-slate-700 uppercase bg-slate-100">
                                <tr>
                                    <th scope="col" class="px-6 py-3">Mã Đèn</th>
                                    <th scope="col" class="px-6 py-3">Tên Mẫu Đèn</th>
                                    <th scope="col" class="px-6 py-3">Người Mượn</th>
                                    <th scope="col" class="px-6 py-3">Dự án</th>
                                    <th scope="col" class="px-6 py-3">Ngày Mượn</th>
                                    <th scope="col" class="px-6 py-3">Ngày Trả Dự Kiến</th>
                                    <th scope="col" class="px-6 py-3">Hành Động</th>
                                </tr>
                            </thead>
                            <tbody id="outstanding-loans-table-body">
                            </tbody>
                        </table>
                    </div>
                </section>
                
                <section id="samples-management">
                     <div class="mb-6">
                        <h2 class="text-xl font-semibold text-slate-900">Danh Sách Toàn Bộ Mẫu Đèn</h2>
                         <p class="text-slate-500 mt-1">Đây là danh sách tổng hợp tất cả các mẫu đèn trong showroom. Bạn có thể tìm kiếm, thêm mới, hoặc thực hiện các thao tác cho mượn/nhận lại từ đây.</p>
                    </div>
                    <div class="flex flex-col sm:flex-row justify-between items-center mb-4 gap-4">
                        <input type="text" id="search-samples" placeholder="Tìm kiếm theo mã hoặc tên đèn..." class="w-full sm:w-1/2 px-4 py-2 border border-slate-300 rounded-lg focus:ring-blue-500 focus:border-blue-500">
                        <button id="btn-add-new" class="w-full sm:w-auto bg-blue-600 text-white font-semibold px-4 py-2 rounded-lg hover:bg-blue-700 transition-colors">Thêm Mẫu Đèn Mới</button>
                    </div>
                    <div class="overflow-x-auto bg-white rounded-lg shadow-sm">
                        <table class="w-full text-sm text-left text-slate-500">
                            <thead class="text-xs text-slate-700 uppercase bg-slate-100">
                                <tr>
                                    <th scope="col" class="px-6 py-3">Hình Ảnh</th>
                                    <th scope="col" class="px-6 py-3">Mã Đèn</th>
                                    <th scope="col" class="px-6 py-3">Tên Mẫu Đèn</th>
                                    <th scope="col" class="px-6 py-3">Trạng Thái</th>
                                    <th scope="col" class="px-6 py-3">Người Mượn Hiện Tại</th>
                                    <th scope="col" class="px-6 py-3">Hành Động</th>
                                </tr>
                            </thead>
                            <tbody id="samples-table-body">
                            </tbody>
                        </table>
                    </div>
                </section>
            </div>

            <div id="content-history" class="tab-content hidden">
                <section id="history-log">
                     <div class="mb-6">
                        <h2 class="text-xl font-semibold text-slate-900">Toàn Bộ Lịch Sử Mượn Trả</h2>
                         <p class="text-slate-500 mt-1">Trang này ghi lại chi tiết tất cả các lượt mượn và trả của từng mẫu đèn. Sử dụng thanh tìm kiếm để tra cứu thông tin nhanh chóng khi cần.</p>
                    </div>
                    <div class="flex justify-between items-center mb-4">
                        <input type="text" id="search-history" placeholder="Tìm kiếm trong lịch sử..." class="w-full sm:w-1/2 px-4 py-2 border border-slate-300 rounded-lg focus:ring-blue-500 focus:border-blue-500">
                    </div>
                     <div class="overflow-x-auto bg-white rounded-lg shadow-sm">
                        <table class="w-full text-sm text-left text-slate-500">
                            <thead class="text-xs text-slate-700 uppercase bg-slate-100">
                                <tr>
                                    <th scope="col" class="px-6 py-3">ID Lượt</th>
                                    <th scope="col" class="px-6 py-3">Mã Đèn</th>
                                    <th scope="col" class="px-6 py-3">Người Mượn</th>
                                    <th scope="col" class="px-6 py-3">Dự án</th>
                                    <th scope="col" class="px-6 py-3">Ngày Mượn</th>
                                    <th scope="col" class="px-6 py-3">Ngày Trả Thực Tế</th>
                                    <th scope="col" class="px-6 py-3">Tình Trạng Khi Trả</th>
                                </tr>
                            </thead>
                            <tbody id="history-table-body">
                            </tbody>
                        </table>
                    </div>
                </section>
            </div>
        </main>
    </div>

    <!-- Modals -->
    <div id="modal-add-sample" class="modal-backdrop fixed inset-0 z-50 flex items-center justify-center p-4 hidden">
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md">
            <div class="p-6 border-b">
                <h3 class="text-lg font-medium">Thêm Mẫu Đèn Mới</h3>
            </div>
            <form id="form-add-sample" class="p-6 space-y-4">
                <div>
                    <label for="new-ma-den" class="block text-sm font-medium text-slate-700">Mã Đèn</label>
                    <input type="text" id="new-ma-den" required class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                <div>
                    <label for="new-ten-mau-den" class="block text-sm font-medium text-slate-700">Tên Mẫu Đèn</label>
                    <input type="text" id="new-ten-mau-den" required class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                 <div>
                    <label for="new-vi-tri" class="block text-sm font-medium text-slate-700">Vị Trí Trưng Bày/Kiểu Đèn</label>
                    <input type="text" id="new-vi-tri" class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500" placeholder="VD: Đèn trần, Đèn bàn, Đèn trang trí...">
                </div>
                <div>
                    <label for="new-hinh-anh-url" class="block text-sm font-medium text-slate-700">URL Hình Ảnh (Tùy chọn)</label>
                    <input type="url" id="new-hinh-anh-url" class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500" placeholder="http://example.com/image.jpg">
                </div>
                <div>
                    <label for="new-mo-ta-san-pham" class="block text-sm font-medium text-slate-700">Mô Tả Sản Phẩm</label>
                    <div class="flex items-center gap-2 mt-1">
                        <textarea id="new-mo-ta-san-pham" rows="4" class="block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"></textarea>
                        <button type="button" id="btn-generate-description" class="bg-purple-600 text-white text-sm font-semibold p-2 rounded-lg hover:bg-purple-700 transition-colors flex-shrink-0 flex items-center justify-center">
                            Tạo Mô Tả ✨
                        </button>
                    </div>
                    <div id="description-loading" class="hidden text-sm text-slate-500 mt-2 flex items-center">
                        <div class="loading-spinner mr-2"></div> Đang tạo mô tả...
                    </div>
                </div>
                <div class="pt-4 flex justify-end gap-3">
                    <button type="button" class="btn-cancel-modal bg-slate-100 text-slate-700 px-4 py-2 rounded-lg hover:bg-slate-200">Hủy</button>
                    <button type="submit" class="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700">Thêm Mới</button>
                </div>
            </form>
        </div>
    </div>
    
    <div id="modal-edit-sample" class="modal-backdrop fixed inset-0 z-50 flex items-center justify-center p-4 hidden">
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md">
            <div class="p-6 border-b">
                <h3 class="text-lg font-medium">Chỉnh Sửa Mẫu Đèn: <span id="edit-item-name" class="font-bold"></span></h3>
            </div>
            <form id="form-edit-sample" class="p-6 space-y-4">
                <div>
                    <label for="edit-ma-den" class="block text-sm font-medium text-slate-700">Mã Đèn</label>
                    <input type="text" id="edit-ma-den" required class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                <div>
                    <label for="edit-ten-mau-den" class="block text-sm font-medium text-slate-700">Tên Mẫu Đèn</label>
                    <input type="text" id="edit-ten-mau-den" required class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                 <div>
                    <label for="edit-vi-tri" class="block text-sm font-medium text-slate-700">Vị Trí Trưng Bày/Kiểu Đèn</label>
                    <input type="text" id="edit-vi-tri" class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500" placeholder="VD: Đèn trần, Đèn bàn, Đèn trang trí...">
                </div>
                <div>
                    <label for="edit-hinh-anh-url" class="block text-sm font-medium text-slate-700">URL Hình Ảnh (Tùy chọn)</label>
                    <input type="url" id="edit-hinh-anh-url" class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500" placeholder="http://example.com/image.jpg">
                </div>
                <div>
                    <label for="edit-mo-ta-san-pham" class="block text-sm font-medium text-slate-700">Mô Tả Sản Phẩm</label>
                    <div class="flex items-center gap-2 mt-1">
                        <textarea id="edit-mo-ta-san-pham" rows="4" class="block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500"></textarea>
                        <button type="button" id="btn-generate-description-edit" class="bg-purple-600 text-white text-sm font-semibold p-2 rounded-lg hover:bg-purple-700 transition-colors flex-shrink-0 flex items-center justify-center">
                            Tạo Mô Tả ✨
                        </button>
                    </div>
                    <div id="description-loading-edit" class="hidden text-sm text-slate-500 mt-2 flex items-center">
                        <div class="loading-spinner mr-2"></div> Đang tạo mô tả...
                    </div>
                </div>
                <div class="pt-4 flex justify-end gap-3">
                    <button type="button" class="btn-cancel-modal bg-slate-100 text-slate-700 px-4 py-2 rounded-lg hover:bg-slate-200">Hủy</button>
                    <button type="submit" class="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700">Lưu Thay Đổi</button>
                </div>
            </form>
        </div>
    </div>

    <div id="modal-loan-sample" class="modal-backdrop fixed inset-0 z-50 flex items-center justify-center p-4 hidden">
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md">
            <div class="p-6 border-b">
                <h3 class="text-lg font-medium">Cho Mượn Mẫu Đèn: <span id="loan-item-name" class="font-bold"></span></h3>
            </div>
            <form id="form-loan-sample" class="p-6 space-y-4">
                <input type="hidden" id="loan-ma-den">
                <div>
                    <label for="loan-nguoi-muon" class="block text-sm font-medium text-slate-700">Người Mượn</label>
                    <input type="text" id="loan-nguoi-muon" required class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                <div>
                    <label for="loan-sdt-email" class="block text-sm font-medium text-slate-700">SĐT/Email Người Mượn</label>
                    <input type="text" id="loan-sdt-email" class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                <div>
                    <label for="loan-du-an" class="block text-sm font-medium text-slate-700">Dự án</label>
                    <input type="text" id="loan-du-an" class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500" placeholder="Tên dự án (nếu có)">
                </div>
                <div>
                    <label for="loan-ngay-tra-du-kien" class="block text-sm font-medium text-slate-700">Ngày Trả Dự Kiến</label>
                    <input type="date" id="loan-ngay-tra-du-kien" required class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                 <div>
                    <label for="loan-tinh-trang-khi-muon" class="block text-sm font-medium text-slate-700">Tình Trạng Khi Mượn</label>
                    <input type="text" id="loan-tinh-trang-khi-muon" value="Tốt, không trầy xước" class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                <div class="pt-4 flex justify-end gap-3">
                    <button type="button" class="btn-cancel-modal bg-slate-100 text-slate-700 px-4 py-2 rounded-lg hover:bg-slate-200">Hủy</button>
                    <button type="submit" class="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700">Xác Nhận Cho Mượn</button>
                </div>
            </form>
        </div>
    </div>

    <div id="modal-return-sample" class="modal-backdrop fixed inset-0 z-50 flex items-center justify-center p-4 hidden">
        <div class="bg-white rounded-lg shadow-xl w-full max-w-md">
            <div class="p-6 border-b">
                <h3 class="text-lg font-medium">Nhận Lại Mẫu Đèn: <span id="return-item-name" class="font-bold"></span></h3>
            </div>
            <form id="form-return-sample" class="p-6 space-y-4">
                <input type="hidden" id="return-ma-den">
                <div>
                    <label for="return-tinh-trang-khi-tra" class="block text-sm font-medium text-slate-700">Tình Trạng Khi Trả</label>
                    <input type="text" id="return-tinh-trang-khi-tra" value="Hoàn hảo, không vấn đề" required class="mt-1 block w-full px-3 py-2 border border-slate-300 rounded-md shadow-sm focus:outline-none focus:ring-blue-500 focus:border-blue-500">
                </div>
                <div class="pt-4 flex justify-end gap-3">
                    <button type="button" class="btn-cancel-modal bg-slate-100 text-slate-700 px-4 py-2 rounded-lg hover:bg-slate-200">Hủy</button>
                    <button type="submit" class="bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700">Xác Nhận Đã Nhận</button>
                </div>
            </form>
        </div>
    </div>


    <script>
        document.addEventListener('DOMContentLoaded', () => {
            
            // --- DATA ---
            let samplesData = [
                { maDen: 'TR001', tenMauDen: 'Đèn trần pha lê tròn', viTri: 'Kệ 1', tinhTrang: 'Tại Showroom', nguoiMuon: '', sdtEmail: '', ngayMuon: '', ngayTraDuKien: '', duAn: '', moTa: 'Một chiếc đèn trần pha lê tròn lộng lẫy, mang đến vẻ sang trọng và đẳng cấp cho không gian.', hinhAnh: 'https://placehold.co/60x60/a78bfa/ffffff?text=TR001' },
                { maDen: 'TR002', tenMauDen: 'Đèn trần gỗ sồi vuông', viTri: 'Kệ 1', tinhTrang: 'Tại Showroom', nguoiMuon: '', sdtEmail: '', ngayMuon: '', ngayTraDuKien: '', duAn: '', moTa: 'Đèn trần gỗ sồi vuông với thiết kế tối giản, tạo cảm giác ấm cúng và gần gũi với thiên nhiên.', hinhAnh: 'https://placehold.co/60x60/8b5cf6/ffffff?text=TR002' },
                { maDen: 'DB001', tenMauDen: 'Đèn bàn đọc sách LED', viTri: 'Khu vực bàn làm việc', tinhTrang: 'Đã Mượn', nguoiMuon: 'Anh Khoa', sdtEmail: '090xxxxxxx', ngayMuon: '2025-06-01', ngayTraDuKien: '2025-06-08', duAn: 'Dự án A', moTa: 'Đèn bàn đọc sách LED hiện đại, cung cấp ánh sáng dịu nhẹ, bảo vệ mắt, lý tưởng cho việc học tập và làm việc.', hinhAnh: 'https://placehold.co/60x60/c084fc/ffffff?text=DB001' },
                { maDen: 'DC001', tenMauDen: 'Đèn cây đứng hiện đại', viTri: 'Góc phòng khách', tinhTrang: 'Tại Showroom', nguoiMuon: '', sdtEmail: '', ngayMuon: '', ngayTraDuKien: '', duAn: '', moTa: 'Chiếc đèn cây đứng với thiết kế thanh lịch, mang đến nguồn sáng ấn tượng và tô điểm cho không gian sống.', hinhAnh: 'https://placehold.co/60x60/e879f9/ffffff?text=DC001' },
                 { maDen: 'TT001', tenMauDen: 'Đèn thả bàn ăn cổ điển', viTri: 'Khu vực bếp', tinhTrang: 'Đang Bảo Trì', nguoiMuon: '', sdtEmail: '', ngayMuon: '', ngayTraDuKien: '', duAn: '', moTa: 'Đèn thả bàn ăn phong cách cổ điển, tạo điểm nhấn ấm cúng và sang trọng cho khu vực dùng bữa.', hinhAnh: 'https://placehold.co/60x60/f0abfc/ffffff?text=TT001' },
            ];

            let historyData = [
                { id: 'LS001', maDen: 'DB001', tenMauDen: 'Đèn bàn đọc sách LED', nguoiMuon: 'Anh Khoa', sdtEmail: '090xxxxxxx', ngayMuon: '2025-06-01', ngayTraDuKien: '2025-06-08', duAn: 'Dự án A', ngayTraThucTe: '', tinhTrangKhiMuon: 'Tốt, không trầy xước', tinhTrangKhiTra: '' },
                { id: 'LS002', maDen: 'DC001', tenMauDen: 'Đèn cây đứng hiện đại', nguoiMuon: 'Chị Lan', sdtEmail: 'lan@email.com', ngayMuon: '2025-05-20', ngayTraDuKien: '2025-05-27', duAn: 'Dự án B', ngayTraThucTe: '2025-05-26', tinhTrangKhiMuon: 'Tốt', tinhTrangKhiTra: 'Hoàn hảo' },
            ];

            // --- UI ELEMENTS ---
            const tabDashboard = document.getElementById('tab-dashboard');
            const tabHistory = document.getElementById('tab-history');
            const contentDashboard = document.getElementById('content-dashboard');
            const contentHistory = document.getElementById('content-history');
            const samplesTableBody = document.getElementById('samples-table-body');
            const outstandingLoansTableBody = document.getElementById('outstanding-loans-table-body'); // New table body
            const historyTableBody = document.getElementById('history-table-body');
            const searchSamplesInput = document.getElementById('search-samples');
            const searchHistoryInput = document.getElementById('search-history');
            
            const btnAddNew = document.getElementById('btn-add-new');
            const modalAddSample = document.getElementById('modal-add-sample');
            const formAddSample = document.getElementById('form-add-sample');

            const modalEditSample = document.getElementById('modal-edit-sample'); // New Edit Modal
            const formEditSample = document.getElementById('form-edit-sample'); // New Edit Form

            const modalLoanSample = document.getElementById('modal-loan-sample');
            const formLoanSample = document.getElementById('form-loan-sample');

            const modalReturnSample = document.getElementById('modal-return-sample');
            const formReturnSample = document.getElementById('form-return-sample');

            const cancelButtons = document.querySelectorAll('.btn-cancel-modal');
            
            const btnGenerateDescription = document.getElementById('btn-generate-description');
            const newMoTaSanPham = document.getElementById('new-mo-ta-san-pham');
            const descriptionLoading = document.getElementById('description-loading');
            const newHinhAnhUrl = document.getElementById('new-hinh-anh-url');

            const btnGenerateDescriptionEdit = document.getElementById('btn-generate-description-edit'); // New for edit modal
            const editMoTaSanPham = document.getElementById('edit-mo-ta-san-pham'); // New for edit modal
            const descriptionLoadingEdit = document.getElementById('description-loading-edit'); // New for edit modal
            const editHinhAnhUrl = document.getElementById('edit-hinh-anh-url'); // New for edit modal
            const editTenMauDen = document.getElementById('edit-ten-mau-den'); // New for edit modal
            const editViTri = document.getElementById('edit-vi-tri'); // New for edit modal
            const editMaDen = document.getElementById('edit-ma-den'); // New for edit modal
            const editItemName = document.getElementById('edit-item-name'); // New for edit modal

            const loanDuAnInput = document.getElementById('loan-du-an');
            
            let statusChart;

            // --- RENDERING FUNCTIONS ---
            const getStatusClass = (status) => {
                switch (status) {
                    case 'Tại Showroom': return 'bg-green-100 text-green-800';
                    case 'Đã Mượn': return 'bg-amber-100 text-amber-800';
                    case 'Đang Bảo Trì': return 'bg-yellow-100 text-yellow-800';
                    default: return 'bg-slate-100 text-slate-800';
                }
            };
            
            const renderSamplesTable = (data = samplesData) => {
                samplesTableBody.innerHTML = '';
                if (data.length === 0) {
                     samplesTableBody.innerHTML = `<tr><td colspan="6" class="text-center py-10 text-slate-500">Không tìm thấy mẫu đèn nào.</td></tr>`;
                     return;
                }
                data.forEach(item => {
                    const statusClass = getStatusClass(item.tinhTrang);
                    const defaultImageUrl = 'https://placehold.co/60x60/f0f0f0/666666?text=No+Img';
                    const imageUrl = item.hinhAnh && item.hinhAnh.startsWith('http') ? item.hinhAnh : defaultImageUrl;

                    const row = document.createElement('tr');
                    row.className = 'bg-white border-b hover:bg-slate-50';
                    row.innerHTML = `
                        <td class="px-6 py-4">
                            <img src="${imageUrl}" alt="${item.tenMauDen}" class="w-16 h-16 object-cover rounded-md" onerror="this.onerror=null;this.src='${defaultImageUrl}';">
                        </td>
                        <td class="px-6 py-4 font-medium text-slate-900 whitespace-nowrap">${item.maDen}</td>
                        <td class="px-6 py-4">${item.tenMauDen}</td>
                        <td class="px-6 py-4">
                            <span class="px-2 py-1 text-xs font-medium rounded-full ${statusClass}">
                                ${item.tinhTrang}
                            </span>
                        </td>
                        <td class="px-6 py-4">${item.nguoiMuon || '---'}</td>
                        <td class="px-6 py-4 flex items-center space-x-2">
                            ${item.tinhTrang === 'Tại Showroom' ? `<button class="btn-loan text-blue-600 hover:text-blue-800 font-semibold" data-ma-den="${item.maDen}">Cho Mượn</button>` : ''}
                            ${item.tinhTrang === 'Đã Mượn' ? `<button class="btn-return text-green-600 hover:text-green-800 font-semibold" data-ma-den="${item.maDen}">Nhận Lại</button>` : ''}
                            <button class="btn-edit text-slate-600 hover:text-slate-800 font-semibold" data-ma-den="${item.maDen}">Sửa</button>
                        </td>
                    `;
                    samplesTableBody.appendChild(row);
                });
            };

            const renderOutstandingLoansTable = () => {
                const loanedItems = samplesData.filter(s => s.tinhTrang === 'Đã Mượn');
                outstandingLoansTableBody.innerHTML = '';
                if (loanedItems.length === 0) {
                     outstandingLoansTableBody.innerHTML = `<tr><td colspan="7" class="text-center py-10 text-slate-500">Không có mẫu đèn nào đang được cho mượn.</td></tr>`;
                     return;
                }
                loanedItems.forEach(item => {
                    const row = document.createElement('tr');
                    row.className = 'bg-white border-b hover:bg-slate-50';
                    row.innerHTML = `
                        <td class="px-6 py-4 font-medium text-slate-900 whitespace-nowrap">${item.maDen}</td>
                        <td class="px-6 py-4">${item.tenMauDen}</td>
                        <td class="px-6 py-4">${item.nguoiMuon || '---'}</td>
                        <td class="px-6 py-4">${item.duAn || '---'}</td>
                        <td class="px-6 py-4">${item.ngayMuon || '---'}</td>
                        <td class="px-6 py-4">${item.ngayTraDuKien || '---'}</td>
                        <td class="px-6 py-4">
                            <button class="btn-return text-green-600 hover:text-green-800 font-semibold" data-ma-den="${item.maDen}">Nhận Lại</button>
                        </td>
                    `;
                    outstandingLoansTableBody.appendChild(row);
                });
            };

            const renderHistoryTable = (data = historyData) => {
                historyTableBody.innerHTML = '';
                 if (data.length === 0) {
                     historyTableBody.innerHTML = `<tr><td colspan="7" class="text-center py-10 text-slate-500">Không có dữ liệu lịch sử.</td></tr>`;
                     return;
                }
                data.forEach(item => {
                    const row = document.createElement('tr');
                    row.className = 'bg-white border-b hover:bg-slate-50';
                    row.innerHTML = `
                        <td class="px-6 py-4 font-medium text-slate-900 whitespace-nowrap">${item.id}</td>
                        <td class="px-6 py-4">${item.maDen}</td>
                        <td class="px-6 py-4">${item.nguoiMuon}</td>
                        <td class="px-6 py-4">${item.duAn || '---'}</td>
                        <td class="px-6 py-4">${item.ngayMuon}</td>
                        <td class="px-6 py-4">${item.ngayTraThucTe || 'Chưa trả'}</td>
                        <td class="px-6 py-4">${item.tinhTrangKhiTra || '---'}</td>
                    `;
                    historyTableBody.appendChild(row);
                });
            };
            
            const renderDashboard = () => {
                const total = samplesData.length;
                const loaned = samplesData.filter(s => s.tinhTrang === 'Đã Mượn').length;
                const available = samplesData.filter(s => s.tinhTrang === 'Tại Showroom').length;
                const maintenance = samplesData.filter(s => s.tinhTrang === 'Đang Bảo Trì').length;

                document.getElementById('total-samples').textContent = total;
                document.getElementById('loaned-samples').textContent = loaned;
                document.getElementById('available-samples').textContent = available;
                
                const ctx = document.getElementById('statusChart').getContext('2d');
                const chartData = {
                    labels: ['Sẵn sàng', 'Đang cho mượn', 'Đang bảo trì'],
                    datasets: [{
                        data: [available, loaned, maintenance],
                        backgroundColor: ['#16a34a', '#d97706', '#facc15'],
                        hoverOffset: 4
                    }]
                };
                
                if (statusChart) {
                    statusChart.data = chartData;
                    statusChart.update();
                } else {
                     statusChart = new Chart(ctx, {
                        type: 'doughnut',
                        data: chartData,
                        options: {
                            responsive: true,
                            maintainAspectRatio: false,
                            plugins: {
                                legend: {
                                    position: 'bottom',
                                }
                            }
                        }
                    });
                }
            };
            
            const updateAll = () => {
                renderSamplesTable();
                renderOutstandingLoansTable(); // Call new render function
                renderHistoryTable();
                renderDashboard();
            }

            // --- EVENT LISTENERS & LOGIC ---
            tabDashboard.addEventListener('click', () => {
                tabDashboard.classList.add('active');
                tabHistory.classList.remove('active');
                contentDashboard.classList.remove('hidden');
                contentHistory.classList.add('hidden');
            });

            tabHistory.addEventListener('click', () => {
                tabHistory.classList.add('active');
                tabDashboard.classList.remove('active');
                contentHistory.classList.remove('hidden');
                contentDashboard.classList.add('hidden');
            });
            
            searchSamplesInput.addEventListener('input', (e) => {
                const searchTerm = e.target.value.toLowerCase();
                const filteredData = samplesData.filter(item => 
                    item.maDen.toLowerCase().includes(searchTerm) ||
                    item.tenMauDen.toLowerCase().includes(searchTerm)
                );
                renderSamplesTable(filteredData);
            });
            
             searchHistoryInput.addEventListener('input', (e) => {
                const searchTerm = e.target.value.toLowerCase();
                const filteredData = historyData.filter(item => 
                    item.id.toLowerCase().includes(searchTerm) ||
                    item.maDen.toLowerCase().includes(searchTerm) ||
                    item.tenMauDen.toLowerCase().includes(searchTerm) ||
                    item.nguoiMuon.toLowerCase().includes(searchTerm) ||
                    (item.duAn && item.duAn.toLowerCase().includes(searchTerm))
                );
                renderHistoryTable(filteredData);
            });

            // Modal handling
            const openModal = (modal) => modal.classList.remove('hidden');
            const closeModal = (modal) => modal.classList.add('hidden');
            
            btnAddNew.addEventListener('click', () => {
                formAddSample.reset(); // Reset form when opening
                newMoTaSanPham.value = ''; // Clear description
                newHinhAnhUrl.value = ''; // Clear image URL
                openModal(modalAddSample);
            });
            cancelButtons.forEach(btn => btn.addEventListener('click', (e) => closeModal(e.target.closest('.modal-backdrop'))));

            // Add new sample
            formAddSample.addEventListener('submit', (e) => {
                e.preventDefault();
                const newSample = {
                    maDen: document.getElementById('new-ma-den').value,
                    tenMauDen: document.getElementById('new-ten-mau-den').value,
                    viTri: document.getElementById('new-vi-tri').value,
                    tinhTrang: 'Tại Showroom',
                    nguoiMuon: '', sdtEmail: '', ngayMuon: '', ngayTraDuKien: '', duAn: '', 
                    moTa: newMoTaSanPham.value,
                    hinhAnh: newHinhAnhUrl.value
                };
                samplesData.unshift(newSample);
                updateAll();
                closeModal(modalAddSample);
                e.target.reset();
            });

            // Gemini API Integration: Generate Product Description for Add Modal
            btnGenerateDescription.addEventListener('click', async () => {
                const tenMauDen = document.getElementById('new-ten-mau-den').value;
                const viTri = document.getElementById('new-vi-tri').value;

                if (!tenMauDen) {
                    newMoTaSanPham.value = 'Vui lòng nhập "Tên Mẫu Đèn" để tạo mô tả.';
                    return;
                }

                descriptionLoading.classList.remove('hidden');
                btnGenerateDescription.disabled = true;
                newMoTaSanPham.value = '';

                const prompt = `Viết một mô tả sản phẩm ngắn gọn, hấp dẫn và sáng tạo cho một mẫu đèn trưng bày trong showroom. Mẫu đèn có tên: "${tenMauDen}". Vị trí trưng bày/kiểu đèn gợi ý: "${viTri}". Hãy tập trung vào việc làm nổi bật vẻ đẹp, chức năng và cảm xúc mà nó mang lại cho không gian.`;

                try {
                    let chatHistory = [];
                    chatHistory.push({ role: "user", parts: [{ text: prompt }] });
                    const payload = { contents: chatHistory };
                    const apiKey = "";
                    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;
                    
                    const response = await fetch(apiUrl, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });
                    const result = await response.json();
                    
                    if (result.candidates && result.candidates.length > 0 &&
                        result.candidates[0].content && result.candidates[0].content.parts &&
                        result.candidates[0].content.parts.length > 0) {
                        const text = result.candidates[0].content.parts[0].text;
                        newMoTaSanPham.value = text;
                    } else {
                        newMoTaSanPham.value = 'Không thể tạo mô tả. Vui lòng thử lại.';
                        console.error('API response error:', result);
                    }
                } catch (error) {
                    newMoTaSanPham.value = 'Lỗi kết nối API. Vui lòng kiểm tra mạng và thử lại.';
                    console.error('Fetch error:', error);
                } finally {
                    descriptionLoading.classList.add('hidden');
                    btnGenerateDescription.disabled = false;
                }
            });

            // Edit Sample (New Feature)
            samplesTableBody.addEventListener('click', e => {
                if(e.target.classList.contains('btn-edit')) {
                    const maDen = e.target.dataset.maDen;
                    const sample = samplesData.find(s => s.maDen === maDen);
                    if (sample) {
                        editItemName.textContent = sample.tenMauDen;
                        editMaDen.value = sample.maDen; // Set original maDen value
                        editTenMauDen.value = sample.tenMauDen;
                        editViTri.value = sample.viTri;
                        editHinhAnhUrl.value = sample.hinhAnh;
                        editMoTaSanPham.value = sample.moTa;
                        openModal(modalEditSample);
                    }
                }
            });

            // Gemini API Integration: Generate Product Description for Edit Modal
            btnGenerateDescriptionEdit.addEventListener('click', async () => {
                const tenMauDen = document.getElementById('edit-ten-mau-den').value;
                const viTri = document.getElementById('edit-vi-tri').value;

                if (!tenMauDen) {
                    editMoTaSanPham.value = 'Vui lòng nhập "Tên Mẫu Đèn" để tạo mô tả.';
                    return;
                }

                descriptionLoadingEdit.classList.remove('hidden');
                btnGenerateDescriptionEdit.disabled = true;
                editMoTaSanPham.value = '';

                const prompt = `Viết một mô tả sản phẩm ngắn gọn, hấp dẫn và sáng tạo cho một mẫu đèn trưng bày trong showroom. Mẫu đèn có tên: "${tenMauDen}". Vị trí trưng bày/kiểu đèn gợi ý: "${viTri}". Hãy tập trung vào việc làm nổi bật vẻ đẹp, chức năng và cảm xúc mà nó mang lại cho không gian.`;

                try {
                    let chatHistory = [];
                    chatHistory.push({ role: "user", parts: [{ text: prompt }] });
                    const payload = { contents: chatHistory };
                    const apiKey = "";
                    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`;
                    
                    const response = await fetch(apiUrl, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify(payload)
                    });
                    const result = await response.json();
                    
                    if (result.candidates && result.candidates.length > 0 &&
                        result.candidates[0].content && result.candidates[0].content.parts &&
                        result.candidates[0].content.parts.length > 0) {
                        const text = result.candidates[0].content.parts[0].text;
                        editMoTaSanPham.value = text;
                    } else {
                        editMoTaSanPham.value = 'Không thể tạo mô tả. Vui lòng thử lại.';
                        console.error('API response error:', result);
                    }
                } catch (error) {
                    editMoTaSanPham.value = 'Lỗi kết nối API. Vui lòng kiểm tra mạng và thử lại.';
                    console.error('Fetch error:', error);
                } finally {
                    descriptionLoadingEdit.classList.add('hidden');
                    btnGenerateDescriptionEdit.disabled = false;
                }
            });


            formEditSample.addEventListener('submit', (e) => {
                e.preventDefault();
                const originalMaDen = document.getElementById('edit-ma-den').dataset.originalMaDen; // Store original maDen
                const newMaDen = document.getElementById('edit-ma-den').value;
                const sampleIndex = samplesData.findIndex(s => s.maDen === originalMaDen);

                if (sampleIndex !== -1) {
                    // Check for duplicate newMaDen, excluding the item being edited itself
                    const isDuplicate = samplesData.some(s => s.maDen === newMaDen && s.maDen !== originalMaDen);
                    if (isDuplicate) {
                        alert('Mã Đèn mới đã tồn tại. Vui lòng chọn Mã Đèn khác.'); // Use alert for simplicity, could be a custom modal
                        return;
                    }

                    // Update maDen in samplesData
                    samplesData[sampleIndex].maDen = newMaDen;
                    samplesData[sampleIndex].tenMauDen = editTenMauDen.value;
                    samplesData[sampleIndex].viTri = editViTri.value;
                    samplesData[sampleIndex].hinhAnh = editHinhAnhUrl.value;
                    samplesData[sampleIndex].moTa = editMoTaSanPham.value;

                    // Update maDen in historyData for all relevant entries
                    historyData.forEach(item => {
                        if (item.maDen === originalMaDen) {
                            item.maDen = newMaDen;
                        }
                    });

                    updateAll();
                    closeModal(modalEditSample);
                }
            });

            // Loan sample
            samplesTableBody.addEventListener('click', e => {
                if(e.target.classList.contains('btn-loan')) {
                    const maDen = e.target.dataset.maDen;
                    const sample = samplesData.find(s => s.maDen === maDen);
                    document.getElementById('loan-item-name').textContent = sample.tenMauDen;
                    document.getElementById('loan-ma-den').value = maDen;
                    document.getElementById('loan-ngay-tra-du-kien').valueAsDate = new Date(Date.now() + 7 * 24 * 60 * 60 * 1000); // Default to 7 days
                    
                    // Clear previous project if any for this loan
                    loanDuAnInput.value = '';
                    openModal(modalLoanSample);
                }
            });

            formLoanSample.addEventListener('submit', (e) => {
                e.preventDefault();
                const maDen = document.getElementById('loan-ma-den').value;
                const sample = samplesData.find(s => s.maDen === maDen);
                const today = new Date().toISOString().split('T')[0];
                
                sample.tinhTrang = 'Đã Mượn';
                sample.nguoiMuon = document.getElementById('loan-nguoi-muon').value;
                sample.sdtEmail = document.getElementById('loan-sdt-email').value;
                sample.ngayMuon = today;
                sample.ngayTraDuKien = document.getElementById('loan-ngay-tra-du-kien').value;
                sample.duAn = document.getElementById('loan-du-an').value; // Save project to sample data

                historyData.unshift({
                    id: 'LS' + String(historyData.length + 1).padStart(3, '0'),
                    maDen: sample.maDen,
                    tenMauDen: sample.tenMauDen,
                    nguoiMuon: sample.nguoiMuon,
                    sdtEmail: sample.sdtEmail,
                    ngayMuon: sample.ngayMuon,
                    ngayTraDuKien: sample.ngayTraDuKien,
                    duAn: sample.duAn, // Save project to history
                    tinhTrangKhiMuon: document.getElementById('loan-tinh-trang-khi-muon').value,
                    ngayTraThucTe: '',
                    tinhTrangKhiTra: ''
                });

                updateAll();
                closeModal(modalLoanSample);
                e.target.reset();
            });

            // Return sample
            // Note: The btn-return listener is now on both samplesTableBody and outstandingLoansTableBody for direct return action
            document.querySelectorAll('#samples-table-body, #outstanding-loans-table-body').forEach(table => {
                table.addEventListener('click', e => {
                    if(e.target.classList.contains('btn-return')) {
                        const maDen = e.target.dataset.maDen;
                        const sample = samplesData.find(s => s.maDen === maDen);
                        document.getElementById('return-item-name').textContent = sample.tenMauDen;
                        document.getElementById('return-ma-den').value = maDen;
                        openModal(modalReturnSample);
                    }
                });
            });


            formReturnSample.addEventListener('submit', e => {
                e.preventDefault();
                const maDen = document.getElementById('return-ma-den').value;
                const sample = samplesData.find(s => s.maDen === maDen);
                const historyItem = historyData.find(h => h.maDen === maDen && h.ngayTraThucTe === '');
                const today = new Date().toISOString().split('T')[0];

                sample.tinhTrang = 'Tại Showroom';
                sample.nguoiMuon = '';
                sample.sdtEmail = '';
                sample.ngayMuon = '';
                sample.ngayTraDuKien = '';
                sample.duAn = ''; // Clear project from sample data when returned

                if (historyItem) {
                    historyItem.ngayTraThucTe = today;
                    historyItem.tinhTrangKhiTra = document.getElementById('return-tinh-trang-khi-tra').value;
                }
                
                updateAll();
                closeModal(modalReturnSample);
                e.target.reset();
            });

            // --- INITIAL LOAD ---
            updateAll();
        });
    </script>

</body>
</html>


# 🐻 Bear Space - Strategic Board Builder

A modern, high-performance web tool for tactical city and structure placement. Design your strategic layouts, manage deployments, and share your configurations with a secure admin system.

![Bear Space Preview](https://placehold.co/1200x600/1e293b/white?text=Bear+Space+Strategic+Board)

## ✨ Features

- **Dynamic Grid Engine**: Configure custom dimensions (Width x Height) for your tactical operations.
- **Drag & Drop Workspace**: Seamlessly place and move cities and structures across the board.
- **Smart City Management**:
  - Automatic re-numbering when cities are removed.
  - Double-click to rename cities directly on the grid.
  - Priority tooltips that stay above all other elements.
- **Strategic Structures**: 
  - **Cities**: Rally, Joiner, and Member types.
  - **Defenses**: HQ, Bear Traps (limit 2), and Flags with 7x7 area of effect visualization.
  - **Environment**: Mountains, Farms, and Statues.
- **Save & Share (Supabase)**:
  - Generate unique, read-only share links for your team.
  - Password-protected admin access for persistence and updates.
- **Visual Precision**: Powered by Vue 3 and Tailwind CSS with a clean, professional aesthetic.

## 🚀 Tech Stack

- **Framework**: [Vue 3 (Composition API)](https://vuejs.org/)
- **Build Tool**: [Vite](https://vitejs.dev/)
- **Styling**: [Tailwind CSS](https://tailwindcss.com/)
- **Database/Auth**: [Supabase](https://supabase.com/)
- **Icons**: Emoji-based for universal compatibility and high performance.

## 🛠️ Getting Started

### Prerequisites
- Node.js (Latest LTS recommended)
- A Supabase project (for save/share functionality)

### Installation

1. **Clone the repository**:
   ```bash
   git clone <repository-url>
   cd bear-whiteout
   ```

2. **Install dependencies**:
   ```bash
   npm install
   ```

3. **Environment Setup**:
   Create a `.env` file in the root directory and add your Supabase credentials:
   ```env
   VITE_SUPABASE_URL=your_supabase_project_url
   VITE_SUPABASE_ANON_KEY=your_supabase_anon_key
   ```

4. **Run Development Server**:
   ```bash
   npm run dev
   ```

## 🎮 How to Use

1. **Placing Elements**: Drag items from the sidebar onto the grid or click an item to place it in the first available slot.
2. **Moving**: Click and drag any item already on the board to reposition it.
3. **Editing Cities**: Double-click a city name on the grid to change it. Press `Enter` to save.
4. **Deleting**: Select an item and press `Delete`/`Backspace`, or use the delete button in the bottom panel.
5. **Saving**: Click "Save & Share", set an admin password, and copy the generated link to share your work.

## 📄 License

This project is specialized for strategic planning and tactical visualizations.

---
*Built with ❤️ for the Bear Space community.*

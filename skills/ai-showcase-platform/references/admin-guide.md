# 管理后台规范

本文档详细描述管理后台的实现规范。

---

## 1. 功能概述

**路径**：`/admin` 及子路由

**功能**：管理员专用的后台管理界面，包含订单管理、用户管理、套餐管理和数据分析。

**访问权限**：仅 `is_admin = true` 的用户可访问

**是否需要登录**：是（必须是管理员账号）

---

## 2. 路由配置

```tsx
// src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import AdminLayout from './layouts/AdminLayout';
import AdminDashboard from './pages/admin/Dashboard';
import AdminOrders from './pages/admin/Orders';
import AdminUsers from './pages/admin/Users';
import AdminProducts from './pages/admin/Products';
import AdminAnalytics from './pages/admin/Analytics';
import ProtectedRoute from './components/ProtectedRoute';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        {/* 公开路由 */}
        {/* ... */}

        {/* 管理后台路由 */}
        <Route
          path="/admin"
          element={
            <ProtectedRoute requireAdmin>
              <AdminLayout />
            </ProtectedRoute>
          }
        >
          <Route index element={<AdminDashboard />} />
          <Route path="orders" element={<AdminOrders />} />
          <Route path="users" element={<AdminUsers />} />
          <Route path="products" element={<AdminProducts />} />
          <Route path="analytics" element={<AdminAnalytics />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

---

## 3. ProtectedRoute 组件

```tsx
// src/components/ProtectedRoute.tsx
import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { supabase } from '../lib/supabase';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requireAdmin?: boolean;
}

const ProtectedRoute = ({ children, requireAdmin = false }: ProtectedRouteProps) => {
  const [loading, setLoading] = useState(true);
  const [authorized, setAuthorized] = useState(false);
  const navigate = useNavigate();

  useEffect(() => {
    const checkAuth = async () => {
      const { data: { session } } = await supabase.auth.getSession();

      if (!session) {
        navigate('/login');
        return;
      }

      if (requireAdmin) {
        // 查询用户是否为管理员
        const { data: profile } = await supabase
          .from('profiles')
          .select('is_admin')
          .eq('id', session.user.id)
          .single();

        if (!profile?.is_admin) {
          alert('无权限访问管理后台');
          navigate('/');
          return;
        }
      }

      setAuthorized(true);
      setLoading(false);
    };

    checkAuth();
  }, [navigate, requireAdmin]);

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center">
        <div className="text-center">
          <i className="ri-loader-4-line text-4xl animate-spin text-gray-400" />
          <p className="mt-4 text-gray-600">验证权限中...</p>
        </div>
      </div>
    );
  }

  return authorized ? <>{children}</> : null;
};
```

---

## 4. AdminLayout 组件

```tsx
// src/layouts/AdminLayout.tsx
import { useState, useEffect } from 'react';
import { Outlet, NavLink, useNavigate } from 'react-router-dom';
import { supabase } from '../lib/supabase';

interface NavItem {
  icon: string;
  label: string;
  path: string;
}

const navItems: NavItem[] = [
  { icon: 'ri-dashboard-line', label: '数据概览', path: '/admin' },
  { icon: 'ri-file-list-3-line', label: '订单管理', path: '/admin/orders' },
  { icon: 'ri-user-line', label: '用户管理', path: '/admin/users' },
  { icon: 'ri-store-2-line', label: '套餐管理', path: '/admin/products' },
  { icon: 'ri-bar-chart-box-line', label: '数据分析', path: '/admin/analytics' },
];

const AdminLayout = () => {
  const [sidebarOpen, setSidebarOpen] = useState(true);
  const [userEmail, setUserEmail] = useState('');
  const navigate = useNavigate();

  useEffect(() => {
    const getUser = async () => {
      const { data: { user } } = await supabase.auth.getUser();
      if (user) {
        setUserEmail(user.email || '');
      }
    };
    getUser();
  }, []);

  const handleLogout = async () => {
    await supabase.auth.signOut();
    navigate('/login');
  };

  return (
    <div className="min-h-screen bg-gray-50 flex">
      {/* 侧边栏 */}
      <aside
        className={`${
          sidebarOpen ? 'w-64' : 'w-20'
        } bg-black text-white transition-all duration-300 flex flex-col`}
      >
        {/* Logo */}
        <div className="p-4 border-b border-gray-800">
          <div className="flex items-center gap-3">
            <div className="w-10 h-10 rounded-lg bg-gradient-to-r from-[#00ff88] to-[#6600ff] flex items-center justify-center">
              <i className="ri-admin-line text-xl text-white" />
            </div>
            {sidebarOpen && <span className="font-bold text-lg">管理后台</span>}
          </div>
        </div>

        {/* 导航 */}
        <nav className="flex-1 py-4">
          {navItems.map((item) => (
            <NavLink
              key={item.path}
              to={item.path}
              end={item.path === '/admin'}
              className={({ isActive }) =>
                `flex items-center gap-3 px-4 py-3 transition-colors ${
                  isActive
                    ? 'bg-gray-800 text-[#00ff88]'
                    : 'text-gray-400 hover:text-white hover:bg-gray-800'
                }`
              }
            >
              <i className={`${item.icon} text-xl`} />
              {sidebarOpen && <span>{item.label}</span>}
            </NavLink>
          ))}
        </nav>

        {/* 底部 */}
        <div className="p-4 border-t border-gray-800">
          <button
            onClick={handleLogout}
            className="flex items-center gap-3 text-gray-400 hover:text-white transition-colors w-full"
          >
            <i className="ri-logout-box-line text-xl" />
            {sidebarOpen && <span>退出登录</span>}
          </button>
        </div>
      </aside>

      {/* 主内容区 */}
      <main className="flex-1 overflow-auto">
        {/* 顶部栏 */}
        <header className="bg-white border-b px-6 py-4 flex items-center justify-between">
          <button
            onClick={() => setSidebarOpen(!sidebarOpen)}
            className="p-2 hover:bg-gray-100 rounded-lg transition-colors"
          >
            <i className="ri-menu-line text-xl" />
          </button>

          <div className="flex items-center gap-4">
            <span className="text-gray-600 text-sm">{userEmail}</span>
            <div className="w-8 h-8 rounded-full bg-gradient-to-r from-[#00ff88] to-[#6600ff] flex items-center justify-center text-white text-sm font-bold">
              A
            </div>
          </div>
        </header>

        {/* 页面内容 */}
        <div className="p-6">
          <Outlet />
        </div>
      </main>
    </div>
  );
};
```

---

## 5. AdminDashboard（数据概览）

```tsx
// src/pages/admin/Dashboard.tsx
import { useEffect, useState } from 'react';
import { Link } from 'react-router-dom';
import { supabase } from '../../lib/supabase';

interface Stats {
  totalUsers: number;
  totalOrders: number;
  totalRevenue: number;
  monthlyNewUsers: number;
}

const AdminDashboard = () => {
  const [stats, setStats] = useState<Stats>({
    totalUsers: 0,
    totalOrders: 0,
    totalRevenue: 0,
    monthlyNewUsers: 0,
  });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchStats = async () => {
      try {
        // 获取用户数
        const { count: userCount } = await supabase
          .from('profiles')
          .select('*', { count: 'exact', head: true });

        // 获取订单数和收入
        const { data: orders } = await supabase
          .from('orders')
          .select('amount, status');

        const paidOrders = orders?.filter((o) => o.status === 'paid') || [];
        const totalRevenue = paidOrders.reduce((sum, o) => sum + (o.amount || 0), 0);

        setStats({
          totalUsers: userCount || 0,
          totalOrders: orders?.length || 0,
          totalRevenue,
          monthlyNewUsers: 0,
        });
      } catch (error) {
        console.error('Error fetching stats:', error);
      } finally {
        setLoading(false);
      }
    };

    fetchStats();
  }, []);

  if (loading) {
    return <div className="text-center py-8">加载中...</div>;
  }

  return (
    <div>
      <h1 className="text-2xl font-bold text-black mb-6">数据概览</h1>

      {/* 统计卡片 */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6 mb-8">
        <StatCard
          icon="ri-user-line"
          label="总用户数"
          value={stats.totalUsers}
          color="#00ff88"
        />
        <StatCard
          icon="ri-file-list-3-line"
          label="总订单数"
          value={stats.totalOrders}
          color="#6600ff"
        />
        <StatCard
          icon="ri-money-dollar-circle-line"
          label="总收入"
          value={`$${stats.totalRevenue.toFixed(2)}`}
          color="#ff00aa"
        />
        <StatCard
          icon="ri-user-add-line"
          label="本月新增"
          value={stats.monthlyNewUsers}
          color="#ffee00"
        />
      </div>

      {/* 快捷操作 */}
      <div className="bg-white rounded-xl p-6 border">
        <h2 className="text-lg font-bold text-black mb-4">快捷操作</h2>
        <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
          <Link
            to="/admin/orders"
            className="p-4 border rounded-lg hover:bg-gray-50 transition-colors"
          >
            <i className="ri-file-list-3-line text-2xl text-gray-600" />
            <p className="mt-2 text-sm font-medium">查看订单</p>
          </Link>
          <Link
            to="/admin/users"
            className="p-4 border rounded-lg hover:bg-gray-50 transition-colors"
          >
            <i className="ri-user-line text-2xl text-gray-600" />
            <p className="mt-2 text-sm font-medium">管理用户</p>
          </Link>
          <Link
            to="/admin/products"
            className="p-4 border rounded-lg hover:bg-gray-50 transition-colors"
          >
            <i className="ri-store-2-line text-2xl text-gray-600" />
            <p className="mt-2 text-sm font-medium">编辑套餐</p>
          </Link>
          <Link
            to="/admin/analytics"
            className="p-4 border rounded-lg hover:bg-gray-50 transition-colors"
          >
            <i className="ri-bar-chart-box-line text-2xl text-gray-600" />
            <p className="mt-2 text-sm font-medium">数据分析</p>
          </Link>
        </div>
      </div>
    </div>
  );
};

// 统计卡片组件
const StatCard = ({
  icon,
  label,
  value,
  color,
}: {
  icon: string;
  label: string;
  value: number | string;
  color: string;
}) => (
  <div className="bg-white rounded-xl p-6 border">
    <div className="flex items-center gap-3">
      <div
        className="w-12 h-12 rounded-lg flex items-center justify-center"
        style={{ backgroundColor: `${color}20` }}
      >
        <i className={icon} style={{ color }} />
      </div>
      <div>
        <p className="text-gray-500 text-sm">{label}</p>
        <p className="text-2xl font-bold text-black">{value}</p>
      </div>
    </div>
  </div>
);
```

---

## 6. AdminOrders（订单管理）

```tsx
// src/pages/admin/Orders.tsx
import { useEffect, useState } from 'react';
import { supabase } from '../../lib/supabase';

interface Order {
  id: string;
  user_email: string;
  product_name: string;
  amount: number;
  status: 'paid' | 'pending' | 'cancelled';
  created_at: string;
}

const AdminOrders = () => {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);
  const [statusFilter, setStatusFilter] = useState('all');
  const [searchQuery, setSearchQuery] = useState('');

  useEffect(() => {
    const fetchOrders = async () => {
      setLoading(true);
      let query = supabase
        .from('orders')
        .select('*')
        .order('created_at', { ascending: false });

      if (statusFilter !== 'all') {
        query = query.eq('status', statusFilter);
      }

      if (searchQuery) {
        query = query.or(`id.ilike.%${searchQuery}%,user_email.ilike.%${searchQuery}%`);
      }

      const { data, error } = await query;

      if (!error && data) {
        setOrders(data);
      }
      setLoading(false);
    };

    fetchOrders();
  }, [statusFilter, searchQuery]);

  const statusBadge = (status: string) => {
    const styles: Record<string, string> = {
      paid: 'bg-green-100 text-green-700',
      pending: 'bg-yellow-100 text-yellow-700',
      cancelled: 'bg-red-100 text-red-700',
    };
    return (
      <span className={`px-2 py-1 rounded-full text-xs font-medium ${styles[status] || ''}`}>
        {status === 'paid' ? '已支付' : status === 'pending' ? '待支付' : '已取消'}
      </span>
    );
  };

  return (
    <div>
      <h1 className="text-2xl font-bold text-black mb-6">订单管理</h1>

      {/* 筛选栏 */}
      <div className="bg-white rounded-xl p-4 border mb-6 flex flex-wrap gap-4 items-center">
        {/* 状态筛选 */}
        <div className="flex items-center gap-2">
          <span className="text-gray-600 text-sm">状态：</span>
          <select
            value={statusFilter}
            onChange={(e) => setStatusFilter(e.target.value)}
            className="px-3 py-2 border rounded-lg focus:outline-none focus:border-black"
          >
            <option value="all">全部</option>
            <option value="paid">已支付</option>
            <option value="pending">待支付</option>
            <option value="cancelled">已取消</option>
          </select>
        </div>

        {/* 搜索框 */}
        <div className="flex-1 flex items-center gap-2">
          <input
            type="text"
            placeholder="搜索订单号或用户邮箱..."
            value={searchQuery}
            onChange={(e) => setSearchQuery(e.target.value)}
            className="flex-1 px-4 py-2 border rounded-lg focus:outline-none focus:border-black"
          />
          <button className="px-4 py-2 bg-black text-white rounded-lg hover:bg-gray-800">
            <i className="ri-search-line" />
          </button>
        </div>
      </div>

      {/* 订单表格 */}
      <div className="bg-white rounded-xl border overflow-hidden">
        <table className="w-full">
          <thead className="bg-gray-50">
            <tr>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">订单号</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">用户</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">商品</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">金额</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">状态</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">时间</th>
            </tr>
          </thead>
          <tbody className="divide-y">
            {loading ? (
              <tr>
                <td colSpan={6} className="px-4 py-8 text-center text-gray-500">
                  加载中...
                </td>
              </tr>
            ) : orders.length === 0 ? (
              <tr>
                <td colSpan={6} className="px-4 py-8 text-center text-gray-500">
                  暂无订单
                </td>
              </tr>
            ) : (
              orders.map((order) => (
                <tr key={order.id} className="hover:bg-gray-50">
                  <td className="px-4 py-3 text-sm font-mono">{order.id.slice(0, 8)}...</td>
                  <td className="px-4 py-3 text-sm">{order.user_email}</td>
                  <td className="px-4 py-3 text-sm">{order.product_name}</td>
                  <td className="px-4 py-3 text-sm font-medium">${order.amount}</td>
                  <td className="px-4 py-3">{statusBadge(order.status)}</td>
                  <td className="px-4 py-3 text-sm text-gray-500">
                    {new Date(order.created_at).toLocaleDateString()}
                  </td>
                </tr>
              ))
            )}
          </tbody>
        </table>
      </div>
    </div>
  );
};
```

---

## 7. AdminUsers（用户管理）

```tsx
// src/pages/admin/Users.tsx
import { useEffect, useState } from 'react';
import { supabase } from '../../lib/supabase';

interface UserProfile {
  id: string;
  email: string;
  nickname: string | null;
  is_admin: boolean;
  created_at: string;
  last_login_at: string | null;
}

const AdminUsers = () => {
  const [users, setUsers] = useState<UserProfile[]>([]);
  const [loading, setLoading] = useState(true);
  const [searchQuery, setSearchQuery] = useState('');

  useEffect(() => {
    const fetchUsers = async () => {
      setLoading(true);
      let query = supabase
        .from('profiles')
        .select('*')
        .order('created_at', { ascending: false });

      if (searchQuery) {
        query = query.or(`email.ilike.%${searchQuery}%,nickname.ilike.%${searchQuery}%`);
      }

      const { data, error } = await query;

      if (!error && data) {
        setUsers(data);
      }
      setLoading(false);
    };

    fetchUsers();
  }, [searchQuery]);

  const handleToggleAdmin = async (userId: string, currentStatus: boolean) => {
    const { error } = await supabase
      .from('profiles')
      .update({ is_admin: !currentStatus })
      .eq('id', userId);

    if (!error) {
      setUsers((prev) =>
        prev.map((u) =>
          u.id === userId ? { ...u, is_admin: !currentStatus } : u
        )
      );
    }
  };

  return (
    <div>
      <h1 className="text-2xl font-bold text-black mb-6">用户管理</h1>

      {/* 搜索框 */}
      <div className="bg-white rounded-xl p-4 border mb-6 flex items-center gap-4">
        <input
          type="text"
          placeholder="搜索邮箱或昵称..."
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          className="flex-1 px-4 py-2 border rounded-lg focus:outline-none focus:border-black"
        />
      </div>

      {/* 用户表格 */}
      <div className="bg-white rounded-xl border overflow-hidden">
        <table className="w-full">
          <thead className="bg-gray-50">
            <tr>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">邮箱</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">昵称</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">角色</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">注册时间</th>
              <th className="px-4 py-3 text-left text-sm font-medium text-gray-600">操作</th>
            </tr>
          </thead>
          <tbody className="divide-y">
            {loading ? (
              <tr>
                <td colSpan={5} className="px-4 py-8 text-center text-gray-500">
                  加载中...
                </td>
              </tr>
            ) : users.length === 0 ? (
              <tr>
                <td colSpan={5} className="px-4 py-8 text-center text-gray-500">
                  暂无用户
                </td>
              </tr>
            ) : (
              users.map((user) => (
                <tr key={user.id} className="hover:bg-gray-50">
                  <td className="px-4 py-3 text-sm">{user.email}</td>
                  <td className="px-4 py-3 text-sm">{user.nickname || '-'}</td>
                  <td className="px-4 py-3">
                    {user.is_admin ? (
                      <span className="px-2 py-1 rounded-full text-xs font-medium bg-gradient-to-r from-[#00ff88] to-[#6600ff] text-white">
                        管理员
                      </span>
                    ) : (
                      <span className="px-2 py-1 rounded-full text-xs font-medium bg-gray-100 text-gray-600">
                        普通用户
                      </span>
                    )}
                  </td>
                  <td className="px-4 py-3 text-sm text-gray-500">
                    {new Date(user.created_at).toLocaleDateString()}
                  </td>
                  <td className="px-4 py-3">
                    <button
                      onClick={() => handleToggleAdmin(user.id, user.is_admin)}
                      className={`px-3 py-1 rounded-lg text-sm font-medium transition-colors ${
                        user.is_admin
                          ? 'bg-red-100 text-red-700 hover:bg-red-200'
                          : 'bg-green-100 text-green-700 hover:bg-green-200'
                      }`}
                    >
                      {user.is_admin ? '移除管理员' : '设为管理员'}
                    </button>
                  </td>
                </tr>
              ))
            )}
          </tbody>
        </table>
      </div>
    </div>
  );
};
```

---

## 8. AdminProducts（套餐管理）

```tsx
// src/pages/admin/Products.tsx
import { useState } from 'react';

interface Product {
  id: string;
  name: string;
  nameEn: string;
  price: number;
  description: string;
  features: string[];
}

// 默认套餐数据（应从配置或数据库读取）
const defaultProducts: Product[] = [
  {
    id: 'starter',
    name: '入门套餐',
    nameEn: 'Starter',
    price: 9.9,
    description: '适合个人用户入门体验',
    features: ['生图 100 张/月', '大模型 1,000 次/月'],
  },
  {
    id: 'pro',
    name: '专业套餐',
    nameEn: 'Pro',
    price: 29.9,
    description: '适合专业用户',
    features: ['全模块无限', '解锁 3D + 视频'],
  },
  {
    id: 'enterprise',
    name: '企业套餐',
    nameEn: 'Enterprise',
    price: 99.9,
    description: '适合企业用户',
    features: ['全模块无限', 'SLA 保障', '私有化部署'],
  },
];

const AdminProducts = () => {
  const [products, setProducts] = useState<Product[]>(defaultProducts);
  const [editingId, setEditingId] = useState<string | null>(null);

  const handleEdit = (id: string) => {
    setEditingId(id);
  };

  const handleSave = async (product: Product) => {
    // TODO: 保存到数据库
    console.log('Saving product:', product);
    setProducts((prev) =>
      prev.map((p) => (p.id === product.id ? product : p))
    );
    setEditingId(null);
  };

  const handleCancel = () => {
    setEditingId(null);
  };

  return (
    <div>
      <h1 className="text-2xl font-bold text-black mb-6">套餐管理</h1>

      {/* 套餐列表 */}
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        {products.map((product) => (
          <div
            key={product.id}
            className="bg-white rounded-xl border overflow-hidden"
          >
            {editingId === product.id ? (
              <ProductForm
                product={product}
                onSave={handleSave}
                onCancel={handleCancel}
              />
            ) : (
              <>
                <div className="p-6">
                  <div className="flex items-center justify-between mb-4">
                    <h3 className="text-lg font-bold">{product.name}</h3>
                    <span className="text-2xl font-bold">${product.price}</span>
                  </div>
                  <p className="text-gray-600 text-sm mb-4">{product.description}</p>
                  <ul className="space-y-2">
                    {product.features.map((feature, idx) => (
                      <li key={idx} className="flex items-center gap-2 text-sm">
                        <i className="ri-check-line text-green-600" />
                        {feature}
                      </li>
                    ))}
                  </ul>
                </div>
                <div className="px-6 py-4 bg-gray-50 border-t flex justify-end">
                  <button
                    onClick={() => handleEdit(product.id)}
                    className="px-4 py-2 bg-black text-white rounded-lg hover:bg-gray-800 text-sm"
                  >
                    编辑
                  </button>
                </div>
              </>
            )}
          </div>
        ))}
      </div>
    </div>
  );
};

// 套餐编辑表单
const ProductForm = ({
  product,
  onSave,
  onCancel,
}: {
  product: Product;
  onSave: (product: Product) => void;
  onCancel: () => void;
}) => {
  const [form, setForm] = useState(product);

  return (
    <div className="p-6">
      <div className="space-y-4">
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">
            套餐名称
          </label>
          <input
            type="text"
            value={form.name}
            onChange={(e) => setForm({ ...form, name: e.target.value })}
            className="w-full px-3 py-2 border rounded-lg focus:outline-none focus:border-black"
          />
        </div>
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">
            价格
          </label>
          <input
            type="number"
            value={form.price}
            onChange={(e) => setForm({ ...form, price: parseFloat(e.target.value) })}
            className="w-full px-3 py-2 border rounded-lg focus:outline-none focus:border-black"
          />
        </div>
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-1">
            描述
          </label>
          <textarea
            value={form.description}
            onChange={(e) => setForm({ ...form, description: e.target.value })}
            rows={2}
            className="w-full px-3 py-2 border rounded-lg focus:outline-none focus:border-black"
          />
        </div>
      </div>
      <div className="mt-6 flex gap-3">
        <button
          onClick={() => onSave(form)}
          className="flex-1 px-4 py-2 bg-black text-white rounded-lg hover:bg-gray-800 text-sm"
        >
          保存
        </button>
        <button
          onClick={onCancel}
          className="flex-1 px-4 py-2 border rounded-lg hover:bg-gray-50 text-sm"
        >
          取消
        </button>
      </div>
    </div>
  );
};
```

---

## 9. AdminAnalytics（数据分析）

```tsx
// src/pages/admin/Analytics.tsx
import { useEffect, useState } from 'react';
import { supabase } from '../../lib/supabase';

const AdminAnalytics = () => {
  const [data, setData] = useState({
    totalRevenue: 0,
    monthlyRevenue: [],
    productDistribution: [],
    recentOrders: [],
  });

  useEffect(() => {
    const fetchAnalytics = async () => {
      // 获取总收入
      const { data: orders } = await supabase
        .from('orders')
        .select('amount, created_at')
        .eq('status', 'paid');

      const paidOrders = orders || [];
      const totalRevenue = paidOrders.reduce((sum, o) => sum + (o.amount || 0), 0);

      // Mock 月度数据（实际应从数据库聚合）
      const monthlyRevenue = [
        { month: '1月', revenue: 1250 },
        { month: '2月', revenue: 2100 },
        { month: '3月', revenue: 1800 },
        { month: '4月', revenue: 2600 },
        { month: '5月', revenue: 3200 },
        { month: '6月', revenue: totalRevenue },
      ];

      // Mock 套餐分布
      const productDistribution = [
        { name: '入门套餐', count: 45, percentage: 38 },
        { name: '专业套餐', count: 52, percentage: 44 },
        { name: '企业套餐', count: 21, percentage: 18 },
      ];

      setData({
        totalRevenue,
        monthlyRevenue,
        productDistribution,
        recentOrders: paidOrders.slice(0, 5),
      });
    };

    fetchAnalytics();
  }, []);

  return (
    <div>
      <h1 className="text-2xl font-bold text-black mb-6">数据分析</h1>

      {/* 总收入 */}
      <div className="bg-gradient-to-r from-[#00ff88] to-[#6600ff] rounded-xl p-6 text-white mb-6">
        <p className="text-white/80">总收入</p>
        <p className="text-4xl font-bold mt-2">${data.totalRevenue.toFixed(2)}</p>
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        {/* 月度收入趋势 */}
        <div className="bg-white rounded-xl border p-6">
          <h3 className="text-lg font-bold mb-4">月度收入趋势</h3>
          <div className="space-y-3">
            {data.monthlyRevenue.map((item, idx) => (
              <div key={idx} className="flex items-center gap-4">
                <span className="w-12 text-sm text-gray-600">{item.month}</span>
                <div className="flex-1 bg-gray-100 rounded-full h-6 overflow-hidden">
                  <div
                    className="h-full bg-gradient-to-r from-[#00ff88] to-[#6600ff] rounded-full"
                    style={{
                      width: `${(item.revenue / 4000) * 100}%`,
                    }}
                  />
                </div>
                <span className="w-20 text-sm font-medium text-right">
                  ${item.revenue}
                </span>
              </div>
            ))}
          </div>
        </div>

        {/* 套餐销量分布 */}
        <div className="bg-white rounded-xl border p-6">
          <h3 className="text-lg font-bold mb-4">套餐销量分布</h3>
          <div className="space-y-4">
            {data.productDistribution.map((item, idx) => {
              const colors = ['#00ff88', '#6600ff', '#ff00aa'];
              return (
                <div key={idx}>
                  <div className="flex justify-between mb-1">
                    <span className="text-sm">{item.name}</span>
                    <span className="text-sm font-medium">{item.count} 单</span>
                  </div>
                  <div className="bg-gray-100 rounded-full h-3 overflow-hidden">
                    <div
                      className="h-full rounded-full"
                      style={{
                        width: `${item.percentage}%`,
                        backgroundColor: colors[idx],
                      }}
                    />
                  </div>
                  <p className="text-xs text-gray-500 mt-1">
                    {item.percentage}% of total
                  </p>
                </div>
              );
            })}
          </div>
        </div>
      </div>
    </div>
  );
};
```

---

## 10. Supabase 表结构

```sql
-- profiles 表（扩展用户信息）
CREATE TABLE profiles (
  id UUID REFERENCES auth.users ON DELETE CASCADE PRIMARY KEY,
  email TEXT,
  nickname TEXT,
  is_admin BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  last_login_at TIMESTAMP WITH TIME ZONE
);

-- 启用 RLS
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- 所有人可读取自己的 profile
CREATE POLICY "Users can view own profile"
  ON profiles FOR SELECT
  USING (auth.uid() = id);

-- 只有管理员可更新所有 profile
CREATE POLICY "Admins can update all profiles"
  ON profiles FOR UPDATE
  USING (
    EXISTS (
      SELECT 1 FROM profiles
      WHERE id = auth.uid() AND is_admin = TRUE
    )
  );

-- 管理员 Cloud Function 设置管理员
CREATE OR REPLACE FUNCTION set_admin(user_id UUID, admin_status BOOLEAN)
RETURNS VOID
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
BEGIN
  -- 检查调用者是否为管理员
  IF NOT EXISTS (
    SELECT 1 FROM profiles
    WHERE id = auth.uid() AND is_admin = TRUE
  ) THEN
    RAISE EXCEPTION 'Unauthorized';
  END IF;

  UPDATE profiles
  SET is_admin = admin_status
  WHERE id = user_id;
END;
$$;
```

---

## 11. 管理员演示账号

### 登录页添加管理员演示入口

```tsx
// src/pages/LoginPage.tsx 中添加
{features.admin && (
  <button
    className="admin-demo-btn mt-4 w-full"
    onClick={() => {
      setEmail('admin@neurahub.com');
      setPassword('admin123456');
    }}
  >
    <i className="ri-admin-line" />
    管理员演示
  </button>
)}
```

### CSS 样式

```css
.admin-demo-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 12px;
  border: 2px solid #000;
  border-radius: 12px;
  font-weight: 600;
  transition: all 0.2s;
}

.admin-demo-btn:hover {
  background: linear-gradient(135deg, #00ff88 0%, #6600ff 100%);
  color: white;
  border-color: transparent;
}
```

---

## 12. 文件结构

```
src/
├── layouts/
│   └── AdminLayout.tsx       # 管理后台布局
├── pages/
│   └── admin/
│       ├── Dashboard.tsx    # 数据概览
│       ├── Orders.tsx        # 订单管理
│       ├── Users.tsx         # 用户管理
│       ├── Products.tsx      # 套餐管理
│       └── Analytics.tsx     # 数据分析
├── components/
│   └── ProtectedRoute.tsx    # 权限保护组件
└── lib/
    └── supabase.ts           # Supabase 客户端
```

---

## 13. 验证清单

- [ ] 非管理员访问 `/admin` 自动跳转首页
- [ ] 侧边栏导航正常，点击切换页面
- [ ] 折叠/展开侧边栏功能正常
- [ ] 退出登录后跳转登录页
- [ ] 订单管理：筛选、搜索功能正常
- [ ] 订单管理：状态徽章显示正确
- [ ] 用户管理：列表显示正常
- [ ] 用户管理：设为/移除管理员功能正常
- [ ] 套餐管理：编辑表单正常
- [ ] 套餐管理：保存后数据更新
- [ ] 数据分析：图表和数据正常显示
- [ ] 移动端侧边栏适配正常

<template>
  <div class="project">
    <Banner
      type="page"
      title="我的项目"
      desc="源于热爱，持续造轮子和练手"
      footer="写 BUG 小能手"
      video="https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/banner.mp4"
    >
      <template #footer-slot>
        <a class="to-github" href="https://github.com/toctong" target="_blank">
          <i class="iconfont icon-github"></i>
          <span>前往 Github</span>
        </a>
      </template>
    </Banner>

    <div class="project-list">
      <div class="project-item">
        <h2 class="name">后台管理系统（Admin Dashboard）</h2>
        <p class="desc">
          基于 NestJS + Vue3 + 原子化 CSS
          的前后端分离后台管理系统，从权限到业务配置都尽量做成「可视化可配置」，方便后续复用到其他项目中。
        </p>
        <p class="desc">
          这个项目更偏向「脚手架 +
          实战模板」：一方面可以当作自己练手和试验新技术的沙盒，另一方面也希望未来能作为中小型后台项目的起点，
          避免每次都从空白项目开始搭权限、菜单和基础设施。
        </p>
        <p class="desc">
          <strong>功能模块一览：</strong><br />
          <strong>· 权限管理：</strong> 登录鉴权、Token 校验，基于角色 / 菜单的路由权限控制。<br />
          <strong>· 用户 &amp; 角色：</strong> 用户信息管理、角色绑定、重置密码、启用 /
          停用等基础操作。<br />
          <strong>· 组织架构：</strong> 部门树、岗位管理，为权限和数据范围打基础。<br />
          <strong>· 菜单 &amp; 按钮：</strong> 菜单层级、按钮级别权限，自定义前端路由 Meta 信息。<br />
          <strong>· 日志审计：</strong> 登录日志、操作日志，方便排查问题和追踪用户行为。<br />
          <strong>· 文件 &amp; 分类：</strong> 文件上传 /
          管理、业务分类管理，支持后续接入对象存储。<br />
          <strong>· 代码集管理：</strong> 常用枚举、字典统一维护，前后端共享一套配置。<br />
        </p>

        <div class="repo-list">
          <a
            class="repo-card"
            href="https://gitee.com/os_nestjs_admin/nest-photo-server"
            target="_blank"
            rel="noreferrer"
          >
            <div class="repo-icon-wrap">
              <img
                class="repo-icon"
                src="https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/nestjs.png"
                alt="NestJS 图标"
              />
            </div>
            <div class="repo-info">
              <div class="repo-name">后端服务 · nest-photo-server</div>
              <div class="repo-meta">NestJS · REST API · 权限 / 日志 / 文件管理</div>
            </div>
          </a>

          <a
            class="repo-card"
            href="https://gitee.com/os_nestjs_admin/arco-admin-ui"
            target="_blank"
            rel="noreferrer"
          >
            <div class="repo-icon-wrap">
              <img
                class="repo-icon"
                src="https://gitee.com/its-liu-xiaodi_admin/my_img/raw/image/vue.png"
                alt="Vue 图标"
              />
            </div>
            <div class="repo-info">
              <div class="repo-name">前端管理端 · arco-admin-ui</div>
              <div class="repo-meta">Vue3 · 组合式 API · 原子化 CSS · Arco Design</div>
            </div>
          </a>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import Banner from "@/components/Banner.vue";
</script>

<style lang="scss" scoped>
.project {
  .banner-page {
    .to-github {
      height: 40px;
      padding: 0 16px;
      display: flex;
      align-items: center;
      justify-content: center;
      border-radius: 50px;
      color: #fff;
      background-color: var(--main-dark-opacity);
      backdrop-filter: blur(20px);
      transition:
        color 0.3s,
        background-color 0.3s;
      .iconfont {
        margin-right: 8px;
        transition: color 0.3s;
      }
      &:hover {
        color: var(--main-card-background);
        background-color: var(--main-color);
        .iconfont {
          color: var(--main-card-background) !important;
        }
      }
    }
  }
  .project-list {
    padding: 2rem 0;

    .project-item {
      .desc {
        margin-bottom: 0.75rem;
      }

      .repo-list {
        display: flex;
        flex-wrap: wrap;
        gap: 12px;
        margin-top: 12px;

        .repo-card {
          display: flex;
          align-items: center;
          padding: 12px 16px;
          border-radius: 14px;
          background-color: var(--main-card-background);
          border: 1px solid rgba(0, 0, 0, 0.04);
          box-shadow: 0 4px 14px rgba(0, 0, 0, 0.05);
          text-decoration: none;
          transition:
            box-shadow 0.2s,
            transform 0.15s,
            border-color 0.2s,
            background-color 0.2s;
          width: calc(50% - 6px);

          .repo-icon-wrap {
            flex: 0 0 auto;
            width: 72px;
            height: 56px;
            border-radius: 12px;
            margin-right: 14px;
            background-color: #fff;
            display: flex;
            align-items: center;
            justify-content: center;

            .repo-icon {
              max-width: 100%;
              max-height: 100%;
              object-fit: contain;
              display: block;
            }
          }

          .repo-info {
            flex: 1;
            display: flex;
            flex-direction: column;
            gap: 4px;

            .repo-name {
              font-size: 0.95rem;
              font-weight: 600;
              color: var(--main-font-color);
            }

            .repo-meta {
              font-size: 0.8rem;
              color: var(--main-font-second-color);
              opacity: 0.85;
            }
          }

          &:hover {
            transform: translateY(-2px);
            box-shadow: 0 8px 24px rgba(0, 0, 0, 0.14);
            border-color: rgba(0, 0, 0, 0.08);
            background-color: rgba(255, 255, 255, 0.98);
          }
        }

        @media (max-width: 768px) {
          .repo-card {
            width: 100%;
            padding: 10px 12px;
            .repo-icon-wrap {
              width: 64px;
              height: 48px;
              margin-right: 10px;
            }
          }
        }
      }
    }
  }
}
</style>

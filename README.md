# FaltuBaat Documentation

> 📚 Documentation website for FaltuBaat - A real-time chat application with video calling and live streaming capabilities.

## 🌐 Live Documentation

Visit the documentation at: **https://cybercoolie.github.io/faltubaatdocs**

## 🚀 Quick Start

### View Online
Simply visit the GitHub Pages URL above to browse the documentation.

### Run Locally

1. **Prerequisites**: Install [Ruby](https://www.ruby-lang.org/) and [Bundler](https://bundler.io/)

2. **Clone the repository**:
   ```bash
   git clone https://github.com/cybercoolie/faltubaatdocs.git
   cd faltubaatdocs
   ```

3. **Install dependencies**:
   ```bash
   bundle install
   ```

4. **Run the development server**:
   ```bash
   bundle exec jekyll serve
   ```

5. **Open in browser**: http://localhost:4000/faltubaatdocs/

## 📁 Structure

```
faltubaatdocs/
├── _config.yml          # Jekyll configuration
├── index.md             # Home page
├── docs/
│   ├── getting-started.md
│   ├── docker.md
│   ├── ec2.md
│   ├── ecs-single.md
│   ├── ecs-multi.md
│   └── eks.md
├── assets/
│   └── images/
└── Gemfile              # Ruby dependencies
```

## 📖 Documentation Topics

| Guide | Description |
|-------|-------------|
| [Getting Started](docs/getting-started.md) | Overview, features, and quick start |
| [Docker](docs/docker.md) | Single & multi-container Docker deployment |
| [EC2](docs/ec2.md) | AWS EC2/VM deployment guide |
| [ECS Single](docs/ecs-single.md) | AWS ECS single container deployment |
| [ECS Multi](docs/ecs-multi.md) | AWS ECS multi-container deployment |
| [EKS](docs/eks.md) | Kubernetes deployment on AWS EKS |

## 🤝 Contributing

1. Fork this repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -m 'Add improvement'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

## 📄 License

This documentation is licensed under the MIT License.

---

**Made with ❤️ by FaltuBaat Team**
